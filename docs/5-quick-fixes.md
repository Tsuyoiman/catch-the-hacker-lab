# 🛠️ Quick fixes

⬅️ [Setup](1-setup.md) · [The lab](2-lab.md) · [README](../README.md)

Find your problem, then do the fix.

---

## Router

| Problem | Fix |
|---|---|
| SecureCRT can't connect to `208.8.8.220` | Router fully booted? Router network adapter = **VMnet8 (NAT)**? From the PC: `ping 208.8.8.220`. Redo setup **S5.1** in the router's VMware window |
| SSH refused / "connection closed" on a new router | In the router's VMware window: `enable` → `configure terminal` → `hostname R-XX` → `ip domain name lab.local` → `crypto key generate rsa modulus 2048` → `end` |
| SSH lands at **`Router>`**, `en` says **"% No password set"** | Your account isn't full admin. Paste the **S5.4** block again (it sets `privilege 15` and `enable secret`). If you can't get in at all, type S5.1 in the router's **VMware window** |
| Paste goes wrong (missing letters, `% Invalid input`) | SecureCRT **Line send delay = 50 ms** (setup **S5.2**), then paste again. Repeating these commands is safe |
| Typed a wrong password in the router's **VMware window**, nothing logged | Use **SecureCRT → SSH**. Only remote (SSH/Telnet) logins are logged |
| Locked out (Step 7) | Wait **60 seconds** |

## Router → Wazuh (no logs arriving)

Check in this order:

| # | Check | Command | ✅ Good |
|---|---|---|---|
| 1 | Router sends to the **right** IP | Router: `show logging \| include Logging to` | `Logging to <MY-WAZUH-IP> … link up` |
| 2 | Router can reach Wazuh | Router: `ping <MY-WAZUH-IP>` | `!!!!!` |
| 3 | Wazuh firewall open | Wazuh: `sudo firewall-cmd --list-ports` | shows `514/tcp` |
| 4 | Wazuh listening | Wazuh: `sudo ss -tulpn \| grep 514` | `LISTEN … :514` |
| 5 | Logs actually arrive | Wazuh: `sudo tcpdump -i any port 514 -A`, then type a wrong password in SecureCRT | text with `LOGIN_FAILED` (press **Ctrl + C** to stop) |

**Wrong IP (e.g. `208.8.8.222`)? Fix it** at `R-XX#`:
```
configure terminal
no logging host 208.8.8.222
logging host <MY-WAZUH-IP> transport tcp port 514
end
write memory
```
**Firewall closed? Open it** on Wazuh:
```bash
sudo firewall-cmd --permanent --add-port=514/tcp ; sudo firewall-cmd --reload
```
> 💡 After restarting your PC, check your Wazuh IP again with `ip a`. If it changed, fix the router's `logging host`.

## Wazuh

| Problem | Fix |
|---|---|
| **Dashboard won't open** ("This site can't be reached") | Open the web port on Wazuh: `sudo firewall-cmd --permanent --add-port=443/tcp` then `sudo firewall-cmd --reload`, and refresh. Also: use **`https://`**, check the IP with `ip a`, and wait **2–3 minutes** after the VM boots |
| "Wazuh dashboard server is not ready yet" | Wait 2–3 min. Still there? `sudo systemctl restart wazuh-indexer`, wait 60 s, `sudo systemctl restart wazuh-dashboard` |
| Dashboard stuck on **"API Connections / Offline"** | Wazuh: `sudo systemctl status wazuh-manager --no-pager`. If **failed**, `ossec.conf` has a mistake: see [Broken ossec.conf](#broken-ossecconf). If **running**: `sudo systemctl restart wazuh-dashboard`, wait 2 min, refresh |
| Dashboard shows login failures that aren't from the router | Those are Wazuh's own Linux logins. Use **Events** and search `full_log:LOGIN_FAILED` |
| Search finds nothing | Time = **Last 15 minutes** / **Last 1 hour**. Try `full_log:*LOGIN_FAILED*` or just `LOGIN_FAILED` |
| **Logins reach Wazuh but the dashboard never updates** (nothing new appears, no matter how many times you log in) | See [Dashboard stops counting](#dashboard-stops-counting-new-events) below |

### Dashboard stops counting new events

Your logins show in `tcpdump` on the Wazuh VM, but the **dashboard never adds new events**. Check these:

**1. Did you turn logging off during the Hide step?** If you ran `no logging host …` (lab Step 3), the router stopped sending. Turn it back on:
```
configure terminal
logging host <MY-WAZUH-IP> transport tcp port 514
end
write memory
```

**2. Did the Wazuh IP change after a restart?** `ip a` on Wazuh. If it's different from the router's `logging host`, fix the router (see [Fix the logging IP](#router--wazuh-no-logs-arriving) above).

**3. Restart the pipeline:**
```bash
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
```
Wait 2 minutes, then test a login again.

### Broken ossec.conf
`wazuh-manager` won't start, with `Error reading XML file 'etc/ossec.conf' (line 0)`. This means a typo in the block you pasted in setup **S4**.

1. Find the broken line:
   ```bash
   (echo "<root>"; sudo cat /var/ossec/etc/ossec.conf; echo "</root>") | xmllint --noout - ; echo "done"
   ```
2. An error like `-:40: … mismatch: allowed-ips … and local_ip` means **line 39** (the number **minus 1**) is wrong.
3. Open it: `sudo nano +39 /var/ossec/etc/ossec.conf`. Every tag must close with the **same name**.
4. Make the block look **exactly** like this (a closing tag with a different name, like `</local_ip>` after `<allowed-ips>`, is the usual cause):
   ```xml
     <remote>
       <connection>syslog</connection>
       <port>514</port>
       <protocol>tcp</protocol>
       <allowed-ips>208.8.8.0/24</allowed-ips>
       <local_ip>0.0.0.0</local_ip>
     </remote>
   ```
5. Save (**Ctrl + O** → Enter → **Ctrl + X**), then:
   ```bash
   sudo systemctl restart wazuh-manager
   sudo systemctl restart wazuh-dashboard
   ```
   ✅ `sudo systemctl status wazuh-manager --no-pager` says **active (running)**.

### Alarms
| Problem | Fix |
|---|---|
| Wazuh won't restart after adding the rules | `NUMBER1` / `NUMBER2` (or `1111` / `2222`) not replaced? On Wazuh 4.12.0: `sudo sed -i 's/NUMBER1/4724/g; s/NUMBER2/4715/g; s/>1111</>4724</g; s/>2222</>4715</g; s/>2501</>4724</g' /var/ossec/etc/rules/local_rules.xml`. Check the errors: `sudo grep -i "error" /var/ossec/logs/ossec.log \| tail -n 5` |
| Brute-force / AI-trick alarm never fires, but single logins show in Events | The failed-login alarm points at the wrong rule. On Wazuh 4.12.0 the live failed login is rule **4724**, not 2501: `sudo sed -i 's/>2501</>4724</g' /var/ossec/etc/rules/local_rules.xml` then `sudo systemctl restart wazuh-manager`. Then do **5 wrong logins within 2 minutes** |
| Alarms added twice (`grep -c "LAB4 ALARM"` shows 6) | Open `sudo nano /var/ossec/etc/rules/local_rules.xml` and delete the **second** `<group name="lab4,cisco_ios,">` … `</group>` block. Save, then `sudo systemctl restart wazuh-manager` |
| Rules don't fire | Test on Wazuh: `sudo /var/ossec/bin/wazuh-logtest` → paste a `LOGIN_FAILED` line **5 times** → the last one should show `id: '100410'`. If not, the number from setup **S9** is wrong |
| Backdoor alarm doesn't fire / no `LOGGEDCMD` lines | The router must **send** config commands to the log. Check `show running-config \| section archive`: it must include `notify syslog contenttype plaintext`. If not: `configure terminal` → `archive` → `log config` → `notify syslog contenttype plaintext` → `end` → `write memory` |

## Wireshark

| Problem | Fix |
|---|---|
| Don't know which adapter | `ipconfig` → the one with **`208.8.8.1`** = **VMware Network Adapter VMnet8** |
| No port-514 packets | Capturing on **VMnet8**? Backup: on Wazuh, `sudo tcpdump -i any port 514 -A` shows the same logs |
| Filter bar turns **red** | Typo. Copy the filter exactly |
| Wireshark shows "RSH" instead of syslog on port 514 | Normal: Wireshark guesses by port number. Use **Follow → TCP Stream** to read it |
| Times don't match Wazuh | **View → Time Display Format → Date and Time of Day** |
