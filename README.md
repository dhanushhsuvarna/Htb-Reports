# HackTheBox Writeups 🔐

A collection of penetration testing writeups and reports for 
HackTheBox machines, documented for learning and reference purposes.

> ⚠️ **Disclaimer:** All writeups are for educational purposes only.
> Testing was performed in a controlled lab environment against 
> intentionally vulnerable machines. Never attempt these techniques 
> against systems you do not have explicit permission to test.

---

## About Me
- **Username:** R00TP1ECE
- **Platform:** [HackTheBox](https://www.hackthebox.com)
- **Focus:** Web exploitation, privilege escalation, CTF challenges, Seasonal Machines HTB

---

## Machines

| # | Machine | OS | Difficulty | Vulnerabilities | Status |
|---|---------|-----|------------|-----------------|--------|
| 1 | [2Million](./machines/2million/report.md) | Linux | Easy | JS Reverse Engineering, Mass Assignment, Command Injection, CVE-2023-0386 | ✅ Completed |

---

## Statistics

| Stat | Value |
|------|-------|
| 🖥️ Machines Completed | 1 |
| 🐧 Linux Machines | 1 |
| 🪟 Windows Machines | 0 |
| ⚡ Easy | 1 |
| 🟡 Medium | 0 |
| 🔴 Hard | 0 |

---

## Attack Techniques Covered

### Web Application
- JavaScript reverse engineering / unpacking
- API enumeration via self-documenting endpoints
- Mass assignment / privilege escalation via API
- Command injection (blind, time-based)
- Base64 / ROT13 encoding/decoding

### Post Exploitation
- Reverse shell stabilization
- Credential harvesting from config files
- Database enumeration (MariaDB/MySQL)
- Password hash cracking (bcrypt)

### Privilege Escalation
- Linux kernel exploitation (CVE-2023-0386)
- OverlayFS SUID bit preservation vulnerability
- SUID binary enumeration

---

## Methodology

Every machine follows the same structured approach:
1. Reconnaissance → nmap, service enumeration
2. Enumeration → web dirs, API endpoints, JS analysis
3. Exploitation → vulnerability identification and exploit
4. Post Exploitation → credential harvesting, lateral movement
5. Privilege Escalation → local privesc to root
6. Reporting → structured findings with remediation


---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and service enumeration |
| Browser DevTools | JavaScript analysis, API interaction |
| de4js | JavaScript unpacking/beautification |
| Burp Suite | HTTP traffic interception and manipulation |
| Netcat (nc) | Reverse shell listener |
| John the Ripper | Password hash cracking |
| hashcat | GPU-accelerated hash cracking |
| GTFOBins | SUID/sudo privilege escalation reference |
| CVE PoC repos | Known vulnerability exploitation |

---

## Resources & References

- [HackTheBox](https://www.hackthebox.com)
- [GTFOBins](https://gtfobins.github.io/) — Unix binary exploitation
- [de4js](https://lelinhtinh.github.io/de4js/) — JavaScript deobfuscator
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — Exploit payloads
- [HackTricks](https://book.hacktricks.xyz/) — Pentesting techniques
- [CVE Details](https://www.cvedetails.com/) — CVE database

---

## Progress

![Machines Completed](https://img.shields.io/badge/Machines%20Completed-2-brightgreen)
![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox)
![Focus](https://img.shields.io/badge/Focus-Linux%20%7C%20Web-blue)

---

*Last updated: October 2026*
