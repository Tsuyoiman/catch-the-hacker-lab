# Part 2: The lab (≈ 100 min)

⬅️ [Setup](1-setup.md) · [README](../README.md) · ➡️ [Mission cards](3-mission-cards.md) · [Detective card](4-detective-card.md) · [Quick fixes](5-quick-fixes.md)

> 🧑‍🏫 **Teacher:** do each step on **your own PC** at the same time, on the projector. You're doing exactly what the students do.

| Step | Time | Activity | Where |
|---|---|---|---|
| **1** | 10 min | 😲 See a password | Your PC |
| **2** | 10 min | 🛠️ Add the alarms | Your PC |
| **3** | 15 min | 🔴 Be the hacker | Your PC |
| **4** | 2 min | 🔄 Move one seat | — |
| **5** | 25 min | 🔵 Be the detective | **Neighbour's PC** |
| **6** | 10 min | 🎭 Reveal + score | Neighbour's PC |
| **7** | 15 min | 🛡️ Defend + clean up | **Back on your PC** |
| **8** | 5 min | ✅ Wrap-up | — |

**Before you start:** VMs on, SecureCRT logged in to the router (`R-XX#`), the Wazuh dashboard open, and Wireshark capturing on **VMnet8**.

---

## Step 1: 😲 See a password (10 min)

🎯 **Teaches:** **Telnet** sends passwords in plain text, and **SSH** encrypts them.

1. At `R-XX#`, paste (turns Telnet **on**):
   ```
   configure terminal
   line vty 0 4
   transport input ssh telnet
   end
   ```
2. SecureCRT → ⚡ → Protocol **Telnet** → Hostname `208.8.8.220` → **Connect** → log in as `admin`.
3. Wireshark → filter `telnet` → **Enter** → right-click any packet → **Follow → TCP Stream**.
   😲 **Your username and password are readable.**
4. Close the stream window. Filter `tcp.port == 22` → right-click → **Follow → TCP Stream**.
   🔒 Only scrambled data. That's SSH.
5. Back in your **SSH** tab, paste (turns Telnet **off**):
   ```
   configure terminal
   line vty 0 4
   transport input ssh
   end
   ```

💬 *Say: "Anyone on the same network could read that password. That's why we only use SSH."*

---

## Step 2: 🛠️ Add the alarms (10 min)

🎯 **Teaches:** a SIEM only knows what its **rules** tell it, and you can write your own (**detection engineering**).

1. In the **Wazuh** SecureCRT tab, open the rules file:
   ```bash
   sudo nano /var/ossec/etc/rules/local_rules.xml
   ```
2. Press **Alt + /** to jump to the end. Paste **all** of [`wazuh/local_rules.xml`](../wazuh/local_rules.xml) at the bottom.
3. Save: **Ctrl + O** → **Enter** → **Ctrl + X**.
4. Put in **your 2 numbers from setup S9**. Replace `1111` and `2222` with your numbers:
   ```bash
   sudo sed -i 's/NUMBER1/1111/g; s/NUMBER2/2222/g' /var/ossec/etc/rules/local_rules.xml
   ```
5. Restart Wazuh:
   ```bash
   sudo systemctl restart wazuh-manager
   sudo systemctl status wazuh-manager --no-pager
   ```
   ✅ **active (running)**. *(Failed? See [Quick fixes](5-quick-fixes.md#alarms).)*

**Your new alarms:**

| Alarm | Fires when | Level |
|---|---|---|
| 🚨 **Brute force** | 5 failed logins within 2 minutes | 10 |
| 🚨 **Backdoor** | A new account with `privilege 15` is created | 12 |
| 🚨 **AI trick** | A login username contains instruction-like words | 10 |

💬 *Say: "You just built the burglar alarm. Now let's see if it catches anyone."*

---

## Step 3: 🔴 Be the hacker (15 min, your own PC)

🎯 **Teaches:** what attackers do: **break in**, **stay** (backdoor) and **hide** (turn off logs).

1. **Wireshark:** click **Capture → Restart** (green fin). This starts a fresh capture, so your Step 1 password isn't in it. **Leave it running.**
2. Take your **secret mission card** ([Mission cards](3-mission-cards.md)). **Don't show anyone.**
3. Do every mission step in **SecureCRT**. **Write the time** of each step on your card.
4. 😲 **Check that your alarms caught you:** Wazuh → **☰ → Threat Hunting → Events** → time **Last 15 minutes** → search:
   ```
   rule.id:(100410 or 100411 or 100412)
   ```
   ✅ **Brute force**, **Backdoor** and **AI trick** alerts. *(If you hid, the alerts from **before** you hid are still there.)*
5. **Don't close anything.** Leave SecureCRT, Wireshark (still capturing) and the Wazuh dashboard **open**.
6. **Turn your mission card face down** next to the keyboard.

---

## Step 4: 🔄 Move one seat (2 min)

Everyone moves **one seat to the right**. The last seat goes to the first.
You're now at your **neighbour's PC**, and you're the **detective** for their router.

---

## Step 5: 🔵 Be the detective (25 min, neighbour's PC)

🎯 **Teaches:** **investigation**: building a timeline from **three** sources of evidence.

1. Take a **[Detective card](4-detective-card.md)** and write the router name (shown in the SecureCRT prompt, e.g. `R-08#`).
2. Use **only the evidence on this PC**:

| Evidence | How to open it |
|---|---|
| 🛡️ **Wazuh** (SIEM) | Dashboard → **Threat Hunting → Events** → time **Last 1 hour** |
| 🦈 **Wireshark** (network) | Click the red **■ Stop** button, then use the filters on the card |
| 🖥️ **Router history** (device) | SecureCRT → SSH to `208.8.8.220` as `admin` → run the commands on the card |

3. Answer all the questions and write the **timeline**.

💬 *Teacher walks around and asks: "How do you know? Show me the evidence."*

---

## Step 6: 🎭 Reveal + score (10 min)

1. The **owner** comes over, turns their mission card face up and reads it out loud.
2. The detective scores themselves (out of **12**, see the detective card).
3. The teacher writes the **top scores** on the board 🏆.

💬 *Discuss: "What did you miss? Which evidence helped most: Wazuh, Wireshark or the router history?"*

---

## Step 7: 🛡️ Defend + clean up (15 min, back on your own PC)

🎯 **Teaches:** **incident response** and **account protection**.

1. Go back to **your own** PC. At `R-XX#`, paste. Replace `<backdoor-name>` with the account from **your** card and `<MY-WAZUH-IP>` with your Wazuh IP:
   ```
   configure terminal
   no username <backdoor-name>
   logging host <MY-WAZUH-IP> transport tcp port 514
   login block-for 60 attempts 3 within 30
   end
   write memory
   ```
2. **Check the backdoor is gone:**
   ```
   show running-config | include username
   ```
   ✅ Only `username admin …`.
3. **Test the protection:** open a new SSH session and type **3 wrong passwords** quickly.
   🚫 The router **blocks all logins for 60 seconds**, even with the correct password.
4. **See the block in your evidence:**

| Where | Search / filter | ✅ You see |
|---|---|---|
| Wazuh | `full_log:QUIET_MODE_ON` | The router reported the block |
| Wireshark | `ip.addr == 208.8.8.220 && tcp.port == 22 && tcp.flags.reset == 1` | The router **rejecting** connections |

5. **Discuss: how do we protect accounts?**

| Measure | Why it helps |
|---|---|
| Strong, unique passwords | Much harder to guess |
| MFA / central login (AAA, RADIUS/TACACS+) | A guessed password alone isn't enough |
| Login rate-limit / lockout (`login block-for`) | Slows brute force. ⚠️ Can also lock out real admins |
| SSH only, never Telnet | Passwords stay encrypted |
| Allow admin logins only from admin IPs (ACL on VTY) | Attackers can't even reach the login |
| Alert on new admin accounts | Backdoors are caught fast |
| Send logs to a SIEM **and alert on silence** | Attacks **and** hiding are noticed |

---

## Step 8: ✅ Wrap-up (5 min)

🧑‍🏫 End with these five points:

1. 🔓 **Telnet shows passwords.** Always use SSH.
2. 🚨 **Rules turn logs into alarms**, and you can write them.
3. 🕵️ Attackers **stay** (backdoors) and **hide** (turn off logs).
4. 🤫 **Missing logs are evidence too.** Silence after activity is suspicious.
5. 🤖 **AI can be tricked** by text inside logs. Always check the evidence.
