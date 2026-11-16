# Lab 4: Catch the Hacker 🕵️

### Brute force · Backdoor accounts · Hiding tracks · Wazuh SIEM · Wireshark

**In one sentence:** you **attack a classmate's router** over the network, then become the **SOC analyst** who **investigates the attack on your own router** using **Wazuh** (SIEM) and **Wireshark** (network capture), and **defends it**.

| | |
|---|---|
| ⏱ **Time** | Setup ≈ 30 min (once) · Lab ≈ 110 min |
| 👤 **Who** | One PC per person. **The teacher does exactly the same steps** on their own PC and projects it |
| 🧰 **Tools** | VMware · Wazuh VM · Cisco CSR1000V router VM · SecureCRT · Wireshark · browser |
| ✅ **Safe** | Everything happens **inside your own virtual lab**. You only "attack" your own router |
| 🎓 **Fits** | CompTIA **Security+ (SY0-701)** · Day 2: *Cyber Threats, Attack Investigation & AI Security* |

---

## 📚 Follow these pages in order

| # | Page | What you do | Time |
|---|---|---|---|
| **1** | **[Setup](docs/1-setup.md)** | Connect the router to Wazuh, paste the router config, open Wireshark | 30 min (once) |
| **2** | **[The lab](docs/2-lab.md)** | Make router reachable → see a password → alarms → attack a classmate → investigate your own router → defend. **Everything is on this page.** | 110 min |
| 🖨️ | [Detective card](docs/4-detective-card.md) | *Optional:* print version | — |
| **3** | **[Quick fixes](docs/5-quick-fixes.md)** 🛠️ | If something doesn't work | — |

---

## 🌍 What is this lab for in the real world?

Every company has **network devices** (routers, switches, firewalls) that run the whole network. Whoever controls them controls the traffic. That makes them a favourite target, and a **Security Operations Centre (SOC)** watches them every day.

This lab copies, on a small scale, what really happens:

| What you do in the lab | What it is in the real world |
|---|---|
| Type wrong passwords at the router | **Brute force / password guessing** against a device's login (MITRE ATT&CK **T1110**) |
| Create a hidden admin account | **Persistence**: attackers add their own admin account so they can come back (**T1136**) |
| Turn off logging and clear the log | **Covering tracks / indicator removal** (**T1070**) |
| Log in with Telnet and read the password in Wireshark | **Network sniffing** of clear-text credentials (**T1040**) |
| Put "ignore previous instructions…" in a username | **Prompt injection** against AI tools that read logs |
| Router sends logs to Wazuh, which raises alerts | **Log collection + SIEM detection**, the daily work of a SOC |
| Investigate a classmate's router | **Incident investigation** by a SOC analyst |
| `login block-for`, deleting the backdoor | **Hardening + incident response** |

### 📰 A real example: hidden admin accounts on Cisco routers
In October 2023, attackers abused a flaw in the web interface of Cisco IOS XE (the same software family as the CSR1000V in this lab). They used it to **create new local accounts with privilege level 15 (full admin)** on exposed devices, for example an account named `cisco_support`. Defenders were told to look for **unexpected or newly created local users** in the device logs. That's exactly the "backdoor account" alarm you build in this lab.

*Sources: [Rapid7 analysis of CVE-2023-20198](https://www.rapid7.com/blog/post/2023/10/17/etr-cve-2023-20198-active-exploitation-of-cisco-ios-xe-zero-day-vulnerability/) · [NVD entry](https://nvd.nist.gov/vuln/detail/CVE-2023-20198)*

### 💼 Jobs that do this every day
- **SOC analyst:** watches the SIEM, investigates alerts (Steps 5–6)
- **Network security engineer:** hardens routers and switches (Step 7)
- **Incident responder:** builds the timeline and cleans up (Steps 5–7)
- **Detection engineer:** writes the rules that create alerts (Step 2)

---

## ⚙️ How the lab works

```
 ┌─────────────────────────────── Your PC ────────────────────────────────┐
 │                                                                        │
 │  SecureCRT ──SSH (port 22)──►  Router R-XX  208.8.8.220                │
 │                                   │  "I saw a failed login!"           │
 │                                   │  sends a log (syslog, TCP 514)     │
 │                                   ▼                                    │
 │  Browser ──https──────────────► Wazuh  <MY-WAZUH-IP>  ── rules ─► 🚨   │
 │                                                                        │
 │  Wireshark listens on VMnet8 (the VMs' network: 208.8.8.0/24)          │
 └────────────────────────────────────────────────────────────────────────┘
```

**The journey of one wrong password:**
1. You type a wrong password in **SecureCRT**.
2. The **router** writes a log line: `%SEC_LOGIN-4-LOGIN_FAILED … [user: root] …`.
3. The router **sends** it to **Wazuh** over the network. You can **see it travel in Wireshark**.
4. **Wazuh** checks it against its **rules**. Too many failures, or a new admin account, raises an 🚨 **alert**.

💬 *Explain it like this: "The router is a security guard who writes every event in a notebook and radios it to the control room. **Wazuh is the control room. Wireshark lets us listen to the radio.**"*

---

## 🧑‍🏫 How to teach it

**Format:** every student works on **their own PC**. Routers are reachable across the classroom network (a bridged adapter — [lab Step 0](docs/2-lab.md#step-0--make-your-router-reachable-10-min-once)), so students **attack each other by IP** instead of swapping seats. The teacher posts a **seat → router IP** list and gives each student a target.

| Step | Students | Teach this |
|---|---|---|
| 0. 🌐 Make router reachable | Add a bridged adapter, give the router a `10.3.3.x` IP | **Networking**: NAT vs bridged |
| 1. 😲 See a password | Log in with Telnet → see the password in Wireshark → compare with SSH | **Encryption**: why we never use Telnet |
| 2. 🛠️ Add the alarms | Paste the Wazuh rules | **Detection engineering**: a SIEM only knows its rules |
| 3. 🔴 Attack a classmate | SSH to a classmate's router IP → brute force → plant a backdoor | **Attacker behaviour**: break in, leave a backdoor |
| 4. 🔵 Investigate your router | A classmate attacked *you* — investigate it on your own Wazuh + Wireshark + router | **Investigation, correlation, timelines** |
| 5. 🛡️ Defend | Remove the backdoor; layer `login block-for` + `access-class` | **Defence in depth** |
| 6. 🔁 Attack again | Re-attack a defended router → blocked | **Proving a control works** |
| 7. 🎭 Reveal + score | Attacker & defender compare papers, score out of 11 | What was caught, what was missed |
| 8. ✅ Wrap-up | 5 takeaways | — |

> ⚠️ **Needs the classroom network to allow PC-to-PC traffic.** Test it (lab Step 0.4). If it's blocked, fall back to each student attacking **their own** router and investigating it.

**Teaching tips**
- 🤫 **Keep each hacker's choices secret.** The surprise at the reveal is what makes it stick.
- ❓ **Don't give detectives the answers.** Ask *"How do you know? Show me the evidence."*
- 🏆 **Use a scoreboard** on the whiteboard. It keeps energy high.
- 🧪 **Do the whole lab once yourself** before class (setup + one mission).

### What students should be able to do afterwards
1. Explain how a log travels from a **device** to a **SIEM**
2. Read Cisco login logs and Wazuh **alert levels**
3. Explain why **Telnet** is dangerous and **SSH** is not
4. **Write a detection rule** and see it fire
5. Recognise **brute force**, **backdoor accounts** and **log tampering**
6. Build an **attack timeline** from Wazuh + Wireshark + device evidence
7. Explain how **AI tools can be tricked** through log data
8. **Harden** a router and recommend account protection

| Security+ SY0-701 | Where |
|---|---|
| 2.4 Indicators of malicious activity | Steps 3, 5 |
| 4.4 Security alerting and monitoring | Steps 2, 5 |
| 4.5 Modify enterprise capabilities (hardening) | Step 7 |
| 4.6 Identity and access management | Step 7 |
| 4.8 Incident response | Steps 5–7 |
| 4.9 Data sources for investigations | Steps 1, 5 |

---

## 📁 Repository files

```
├── README.md                     ← you are here (overview, real world, how to teach)
├── docs/
│   ├── 1-setup.md                ← Part 1: setup (once)
│   ├── 2-lab.md                  ← Part 2: the lab, step by step
│   ├── 4-detective-card.md       ← detective questions + scoring (print)
│   ├── 5-quick-fixes.md          ← troubleshooting
│   ├── practice-incident.md      ← optional extra: prepared Windows incident
│   ├── index.html                ← practice SIEM console for the optional extra
│   └── extras/old-router-lab-guide.md  ← previous version of this lab
├── wazuh/local_rules.xml         ← the alarm rules (Step 2)
└── data/, tools/                 ← files for the optional extra
```
