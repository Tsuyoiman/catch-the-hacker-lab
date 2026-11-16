# Part 1: Setup (do this once, ≈ 30 min)

⬅️ [Back to README](../README.md) · ➡️ Next: [The lab](2-lab.md)

> 🧑‍🏫 Teacher and students do **exactly the same** setup, each on their own PC.

---

## Your addresses

| Device | IP / name | Example |
|---|---|---|
| Your PC on the classroom network | your PC's normal IP | `192.168.100.7/24` |
| Your PC on the VMs' network (VMnet8) | `208.8.8.1` | `208.8.8.1/24` |
| VMware NAT gateway | `208.8.8.2` | |
| Router VM (CSR1000V) | `208.8.8.220` | |
| Your Wazuh VM | **`<MY-WAZUH-IP>`** | `208.8.8.140` |
| Your router's name | **`R-XX`** = your seat number | `R-07` |

> ⚠️ Replace every `<…>` with your real value. **Passwords come from your teacher.** Everyone uses the **same class router password**, so the detective can log in to your router in Step 5.

---

## S1. Put both VMs on VMnet8 (NAT)

🎯 Both VMs must be on the **same virtual network**, or the router can't send logs to Wazuh.

1. VMware → **Edit → Virtual Network Editor** → select **VMnet8**:
   - Type: **NAT**
   - ✅ **Use local DHCP service to distribute IP addresses**
   - Subnet IP: **208.8.8.0**, Subnet mask: **255.255.255.0**
2. Right-click the **Wazuh VM** → **Settings → Network Adapter** → **Custom: VMnet8 (NAT)** → **OK**.
3. Right-click the **Router VM (CSR1000V)** → **Settings → Network Adapter** → **Custom: VMnet8 (NAT)** → **OK**.

---

## S2. Start Wazuh and find its IP

1. Right-click the **Wazuh VM** → **Power → Start Up Guest**. Wait for the login prompt.
2. Log in as `root` and type:
   ```bash
   ip a
   ```
3. Find the address starting with **`208.8.8.`** and **write it down**. This is **`<MY-WAZUH-IP>`**.

---

## S3. Connect to Wazuh with SecureCRT

Copy-paste works much better in SecureCRT than in the VMware window.

1. SecureCRT → ⚡ **Quick Connect** (**Alt + Q**).
2. Protocol **SSH2** · Hostname **`<MY-WAZUH-IP>`** · Port **22** · Username **`root`** → **Connect**.
3. Click **Accept** for the host key, then type the root password.

---

## S4. Let Wazuh accept router logs (`ossec.conf`)

🎯 `ossec.conf` is Wazuh's main settings file. You'll add **one block** that says: *"Listen on port 514 for logs from devices on 208.8.8.x."*

**1. Open the file:**
```bash
sudo nano /var/ossec/etc/ossec.conf
```

**2. Jump to the end:** press **Alt + /**. The last line is `</ossec_config>`.

**3. Put the cursor on the empty line *above* the last `</ossec_config>`, then right-click → Paste:**
```xml
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>tcp</protocol>
    <allowed-ips>208.8.8.0/24</allowed-ips>
    <local_ip>0.0.0.0</local_ip>
  </remote>
```

✅ The end of the file must look **exactly** like this:
```xml
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>tcp</protocol>
    <allowed-ips>208.8.8.0/24</allowed-ips>
    <local_ip>0.0.0.0</local_ip>
  </remote>

</ossec_config>
```

| Line | Meaning |
|---|---|
| `<connection>syslog</connection>` | Accept **syslog**, the format routers use to send logs |
| `<port>514</port>` | Listen on port **514** (standard syslog port) |
| `<protocol>tcp</protocol>` | Use **TCP**, the same as the router |
| `<allowed-ips>208.8.8.0/24</allowed-ips>` | Only accept logs from the **VMnet8** network (your router is `.220`) |
| `<local_ip>0.0.0.0</local_ip>` | Listen on **all** of Wazuh's addresses |

> ⚠️ **Careful**
> - **Don't delete** any other `<remote>` block in the file (e.g. one with port `1514`). Wazuh needs it.
> - Every tag must **close with the same name**: `<allowed-ips>…</allowed-ips>`. A wrong closing tag stops Wazuh from starting.

**4. Save and exit:** **Ctrl + O** → **Enter** → **Ctrl + X**.

**5. Check the file has no mistakes:**
```bash
(echo "<root>"; sudo cat /var/ossec/etc/ossec.conf; echo "</root>") | xmllint --noout - ; echo "done"
```
✅ Only **`done`** appears. *(An error? See [Quick fixes](5-quick-fixes.md#broken-ossecconf).)*

**6. Open the firewall (514 = router logs, 443 = dashboard web page), then restart Wazuh:**
```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```
✅ **active (running)**. *(`FirewallD is not running` is fine: there's no firewall to open.)*

**7. Check Wazuh is listening:**
```bash
sudo ss -tulpn | grep 514
```
✅ A line with **`LISTEN`** and **`:514`**.

---

## S5. Set up the router (one copy-paste)

### 5.1 Turn on the router and give it its IP (first time only)
1. Right-click the **Router VM (CSR1000V)** → **Power → Start Up Guest**. Wait for `Router>`.
2. **In the router's VMware window**, type these lines by hand (replace `R-XX` with your seat number and `<router-password>` with the class password). The VMware window can't paste, and this is just enough for SecureCRT to connect:
   ```
   enable
   configure terminal
   interface GigabitEthernet1
   ip address 208.8.8.220 255.255.255.0
   no shutdown
   exit
   username admin privilege 15 secret <router-password>
   hostname R-XX
   ip domain name lab.local
   crypto key generate rsa modulus 2048
   line vty 0 4
   login local
   transport input ssh
   end
   ```
   *(If `crypto key generate` says keys already exist and asks to replace them, type `yes`. On older routers where `ip domain name` is rejected, use `ip domain-name lab.local` instead.)*

### 5.2 Set SecureCRT to paste slowly (once)
Routers can drop letters if text arrives too fast.
SecureCRT → **Options → Session Options → Terminal → Emulation → Advanced** (or **Paste delay**) → set **Line send delay** to **50** ms → **OK**.

### 5.3 SSH to the router
SecureCRT → ⚡ → **SSH2** · `208.8.8.220` · Port **22** · Username **`admin`** → **Connect** → **Accept** the host key → type the router password.
✅ You're at **`Router#`** (with **`#`**).

### 5.4 Paste the full router config
**1. Fill in your values.** Copy the block below into **Notepad** and replace:

| Replace | With | Example |
|---|---|---|
| `R-XX` | Your seat number | `R-07` |
| `<MY-WAZUH-IP>` | Your Wazuh IP from S2 | `208.8.8.140` |
| `<router-password>` | Class router password | *(from the teacher)* |
| `<enable-password>` | Class enable password | *(from the teacher)* |

**2. Copy from Notepad → SecureCRT → right-click → Paste:**
```
configure terminal
hostname R-XX
username admin privilege 15 secret <router-password>
enable secret <enable-password>
interface GigabitEthernet1
ip address 208.8.8.220 255.255.255.0
no shutdown
exit
line vty 0 4
login local
transport input ssh
exit
logging on
logging buffered 16384 informational
logging trap informational
logging source-interface GigabitEthernet1
logging host <MY-WAZUH-IP> transport tcp port 514
login on-failure log
login on-success log
archive
log config
logging enable
logging size 200
notify syslog contenttype plaintext
hidekeys
exit
exit
end
write memory
```
✅ The prompt changes to **`R-XX#`**, and the last line says **`[OK]`**.

**What each part does:**

| Lines | What they do |
|---|---|
| `hostname R-XX` | Names your router. The prompt becomes `R-XX#` |
| `username admin privilege 15 …` | Admin account. **SSH lands straight at `R-XX#`** |
| `enable secret …` | Password for `enable` |
| `interface … 208.8.8.220` | The router's IP |
| `line vty … transport input ssh` | Allows **SSH** logins (encrypted) |
| `logging buffered …` | Keeps recent logs in the router's memory (`show logging`) |
| `logging host <MY-WAZUH-IP> …` | **Sends logs to your Wazuh** |
| `login on-failure log` / `on-success log` | Records **every login**, failed and successful |
| `archive … log config … hidekeys` | Records **every config command** and hides passwords as `*****` |
| `notify syslog contenttype plaintext` | **Sends** those recorded commands to the log and to Wazuh. Without it, the backdoor alarm can't work |
| `write memory` | Saves everything (survives a reboot) |

---

## S6. Check the router sends logs

At `R-XX#`:
```
ping <MY-WAZUH-IP>
show logging | include Logging to
```
✅ `!!!!!`
✅ **`Logging to <MY-WAZUH-IP> (tcp port 514 …) … link up`**

*(Says `208.8.8.222`, or `link down`? See [Quick fixes](5-quick-fixes.md).)*

---

## S7. Open your Wazuh dashboard

1. Browser → **`https://<MY-WAZUH-IP>`** (e.g. `https://208.8.8.140`).
2. Certificate warning → **Advanced → Continue**.
3. Log in with the dashboard account from your teacher.

✅ The **Overview** page with alert counts.

> ⚠️ **Page doesn't open** ("This site can't be reached")? The firewall is blocking the web page. On Wazuh, type:
> ```bash
> sudo firewall-cmd --permanent --add-port=443/tcp
> sudo firewall-cmd --reload
> ```
> Then refresh the browser. *(Stuck on "API Connections / Offline"? See [Quick fixes](5-quick-fixes.md#wazuh).)*

---

## S8. Wireshark on VMnet8

1. On your PC: **Command Prompt** → `ipconfig` → find:
   ```
   Ethernet adapter VMware Network Adapter VMnet8:
      IPv4 Address. . . : 208.8.8.1
   ```
2. **Wireshark** → scroll the interface list → double-click **VMware Network Adapter VMnet8**.
   *(Not Wi-Fi, and not the other VMnet adapters. Your Wi-Fi/LAN `192.168.100.x` is a different network.)*
3. Filter bar → `tcp.port == 514` → **Enter**.
4. SecureCRT → new SSH to `208.8.8.220` → type a **wrong password** once.
   ✅ Packets from **208.8.8.220 → `<MY-WAZUH-IP>`**: the router's log travelling to Wazuh.

---

## S9. Find the 2 rule numbers for your alarms

🎯 Your alarms (Step 2 of the lab) build on rules Wazuh already has. You need their **ID numbers**, which can differ between Wazuh versions. You only do this once.

### 🤔 What is this ID, and why do we copy it?

Wazuh comes with **thousands of built-in rules**, and **each rule has an ID number**. When a log arrives, Wazuh checks it against these rules. The rule that matches **recognises** what the log is. For example: *"this is a failed login"*.

Your new alarms **don't start from zero**. They **build on top of** the built-in rule that already recognised the log:

```
 Router log arrives:  "%SEC_LOGIN-4-LOGIN_FAILED ..."
        │
        ▼
 Built-in rule  ID 4724  →  "This is a failed login"          ← the ID you copy
        │
        ▼  (your alarm says: "only look at logs that matched rule 4724")
 YOUR alarm  ID 100410  →  "5 of them in 2 minutes = 🚨 BRUTE FORCE"
```

| You copy the ID of… | Your alarm uses it to say… |
|---|---|
| the rule that recognises a **failed login** (Number 1) | *"Count failed logins. 5 in 2 min = brute force"* and *"Check the username for AI tricks"* |
| the rule that recognises a **config command** (Number 2) | *"Check config commands. A new `privilege 15` user = backdoor"* |

💡 *Analogy: the built-in rule is the guard who says "that's a failed login". Your alarm is the supervisor who says "if the guard reports 5 of those in 2 minutes, sound the alarm!" The supervisor needs to know **which guard** to listen to, and that's the ID number.*

⚠️ That's why the number must be **exactly** the one Wazuh shows. A made-up number (like `1111`) points to a guard who doesn't exist, so your alarm never hears anything, and Wazuh may refuse to start.

### ✅ On Wazuh 4.12.0 (our class version) the numbers are already known:

| Number | Rule ID | What it recognises |
|---|---|---|
| **Number 1** | **4724** | a **failed login** to the router (`LOGIN_FAILED`) |
| **Number 2** | **4715** | a **config command** on the router (`LOGGEDCMD`) |

**If you're on Wazuh 4.12.0, you don't need to look anything up. Just use 4724 and 4715 in [Step 2 of the lab](2-lab.md#step-2--add-the-alarms-10-min).**

> ℹ️ **You only need 2 numbers for the alarms:** the **failed login** (4724) and the **config command** (4715). The **successful login** rule (`4722`) is not used by any alarm — successful logins show up in the dashboard on their own.

### 🔎 Only if your Wazuh version is different: find your own numbers

There are two ways. **Method A (dashboard)** is the easiest and most reliable. **Method B (`wazuh-logtest`)** shows the Phase 1–3 breakdown if you want to see how Wazuh decodes a log.

**First, make real events.** In SecureCRT:
- open a new SSH session to `208.8.8.220` and type **one wrong password** (this makes a `LOGIN_FAILED`), then
- log in correctly and run a **config command** (this makes a `LOGGEDCMD`):
  ```
  configure terminal
  username testcmd privilege 15 secret Test123
  end
  ```

#### Method A — read the rule ID in the dashboard (recommended)
Wazuh → **Threat Hunting → Events** → time **Last 15 minutes**:
- Search `full_log:LOGIN_FAILED` → click **▸** to open the event → read **`rule.id`**. ✍️ **Number 1 (failed login) = ____**
- Search `full_log:LOGGEDCMD` → open the event → read **`rule.id`**. ✍️ **Number 2 (config command) = ____**

> ⚠️ **`full_log:LOGGEDCMD` shows nothing?** A `LOGGEDCMD` event only exists after you **type a config command** on a router that is set to **send** config commands. Check both:
> 1. **You ran a config command** — e.g. the `username testcmd …` above. Just logging in does **not** create one.
> 2. **The router sends them** — on the router, `show running-config | section archive` must include **`notify syslog contenttype plaintext`**. If it's missing, paste:
>    ```
>    configure terminal
>    archive
>    log config
>    logging enable
>    notify syslog contenttype plaintext
>    hidekeys
>    end
>    write memory
>    ```
> Then run the `username testcmd …` command again and search once more.
>
> 🧹 **Clean up the test account** when done: `configure terminal` → `no username testcmd` → `end`.

#### Method B — see the Phases with `wazuh-logtest`
This is the tool that shows **Phase 1 → 2 → 3**. In the **Wazuh** SecureCRT tab:
```bash
sudo /var/ossec/bin/wazuh-logtest
```
Wait for `Type one log per line`, then **paste a real log line** (copy it from the router with `show logging | include LOGIN_FAILED`, starting from the **sequence number**, e.g. `165: *Nov 16 ...`):
```
165: *Nov 16 18:11:50.811: %SEC_LOGIN-4-LOGIN_FAILED: Login failed [user: admin] [Source: 208.8.8.1] [localport: 22] [Reason: Login Authentication Failed] at 18:11:50 UTC Mon Nov 16 2026
```
Read the output:
```
**Phase 1: Completed pre-decoding.      ← Wazuh splits the raw log
**Phase 2: Completed decoding.          ← it recognises "cisco-ios", mnemonic LOGIN_FAILED
**Phase 3: Completed filtering (rules).
        id: '4724'        ◄── THIS is Number 1
        level: '9'
        description: 'Cisco IOS: Failed login to the router'
```
✍️ **Number 1 (failed login) = the `id` in Phase 3.** Then paste a `LOGGEDCMD` line the same way for **Number 2**. Press **Ctrl + C** to exit.

> ⚠️ **Important:** paste a **real line with the `165: *Nov 16 ...` prefix**, exactly as the router sends it. A hand-typed line **without** the prefix can match a *different* rule and give you the wrong number (we hit this: a clean line gave `2501`, but the real logs are `4724`).

**Finally, clean up the test account:**
```
configure terminal
no username testcmd
end
```

---

✅ **Setup done!** ➡️ Continue to **[The lab](2-lab.md)**.
