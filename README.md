# Home Lab 1 — Penetration Test of Metasploitable 2

A hands-on penetration testing lab built from scratch in an isolated virtual environment. I set up a Kali Linux attacker and a vulnerable Metasploitable 2 target, then carried out a full attack lifecycle — reconnaissance, enumeration, exploitation, and post-exploitation — gaining full root control of the target, and documented the findings with remediation guidance in a professional-style report.

## 🧰 Environment
- **Hypervisor:** Oracle VirtualBox (on Windows 11)
- **Attacker:** Kali Linux — `10.0.0.10`
- **Target:** Metasploitable 2 — `10.0.0.20`
- **Network:** Isolated VirtualBox internal network (no internet exposure)
- **Tools:** Nmap, Metasploit Framework, Netcat

## 🎯 What I did
| Phase | Summary |
|-------|---------|
| Reconnaissance | Scanned the target with Nmap, mapped 23 open ports |
| Enumeration | Identified exact service versions to find known vulnerabilities |
| Exploitation | Gained a **root** shell via the Samba `usermap_script` vulnerability (CVE-2007-2447) |
| Post-Exploitation | Demonstrated impact by extracting the system password hash file (`/etc/shadow`) |

## 📄 Full write-up
See **[pentest-report.md](pentest-report.md)** for the complete report — methodology (mapped to MITRE ATT&CK), findings with severity ratings, evidence, and remediation recommendations.

## 📸 Evidence
Screenshots are in the [`screenshots/`](screenshots/) folder.

---
*This lab was performed against a deliberately vulnerable machine that I own, inside an isolated network. No production or third-party systems were involved.*
