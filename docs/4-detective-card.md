# 🔵 Detective card (print one per student)

⬅️ [The lab](2-lab.md) · [README](../README.md)

---

> ### 🔵 DETECTIVE CARD
> **I'm investigating router:** R-____ **Detective name:** ______________
>
> Use **only the evidence on this PC**:
> - 🛡️ **Wazuh:** Threat Hunting → **Events** → time **Last 1 hour**
> - 🦈 **Wireshark:** click **■ Stop**, then use the filters below
> - 🖥️ **Router:** SecureCRT → SSH2 `208.8.8.220` as `admin` → run the commands below

---

## The questions

| # | Question | 🛡️ Wazuh search | 🦈 Wireshark filter | 🖥️ Router command | ✍️ My answer |
|---|---|---|---|---|---|
| **1** | Which **usernames** were tried? How many wrong tries? When? | `full_log:LOGIN_FAILED` | `frame contains "LOGIN_FAILED"` | `show logging \| include LOGIN_FAILED` | |
| **2** | Did the attacker **get in**? At what time? | `full_log:LOGIN_SUCCESS` | `frame contains "LOGIN_SUCCESS"` | `show logging \| include LOGIN_SUCCESS` | |
| **3** | Was a **backdoor account** created? What's its **name**? | `rule.id:100411` | `frame contains "username"` → right-click → **Follow → TCP Stream** | `show running-config \| include username` | |
| **4** | Did they try to **trick an AI**? Which username? | `rule.id:100412` | `frame contains "ignore" or frame contains "benign" or frame contains "pentest" or frame contains "false-positive" or frame contains "authorised"` | — | |
| **5** | Did they **hide** (turn off logging)? **YES / NO**. When? | `full_log:LOGIN*`, sorted by time. Do the events **stop suddenly**? | `tcp.port == 514`: time of the **last** packet | `show archive log config all` → look for `no logging host` | |
| **6** | If they hid, were they **still active** after? | *(Wazuh is blind now)* | `ip.addr == 208.8.8.220 && tcp.port == 22`: any SSH traffic **after** the last port-514 packet? | `show archive log config all` → commands after `no logging host` | |

> 💡 **Tips**
> - In Wazuh, click **▸** next to an event to read `full_log`. The username is inside `[user: …]`.
> - In Wireshark, **View → Time Display Format → Date and Time of Day** to see clock times.
> - `show archive log config all` lists **every config command** typed on the router, with the user. It survives `clear logging`. 😲
> - If the attacker hid, `show logging` may be **empty** (they ran `clear logging`). Use Wireshark and the archive instead.

---

## My timeline

| Time | What happened | Evidence (Wazuh / Wireshark / Router) |
|---|---|---|
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |

---

## Scoring (fill in at the reveal)

| Found correctly | Points | ✍️ |
|---|---|---|
| Usernames tried + number of wrong tries | 2 | |
| Time of the attack | 1 | |
| Successful login (and time) | 1 | |
| **Backdoor account name** | 3 | |
| AI-trick username | 2 | |
| **Hide: YES or NO**, with evidence | 3 | |
| **Total** | **12** | |

🏆 **10–12** Senior SOC analyst · **7–9** SOC analyst · **4–6** Junior analyst · **0–3** Keep training!
