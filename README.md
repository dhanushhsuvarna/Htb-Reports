# HackTheBox Writeups 🔐

A collection of penetration testing writeups and reports for HackTheBox machines, documented for learning and reference purposes.

> ⚠️ **Disclaimer:** All writeups are for educational purposes only. Testing was performed in a controlled lab environment against intentionally vulnerable machines. Never attempt these techniques against systems you do not have explicit permission to test.

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
| 2 | [Orion](./machines/orion/report.md) | Linux | Very Easy | Craft CMS Exploitation, Credential Disclosure, Password Cracking, Password Reuse, Telnet Privilege Escalation (CVE-2026-24061) | ✅ Completed |
| 3 | [Abducted](./machines/abducted/report.md) | Linux | Medium | Symbolic-link abuse / file handling (per notes); verify exact chain in report | ✅ Completed |
| 4 | [Principal](./machines/principal/report.md) | Easy | Not specified | Not provided in current context | ✅ Completed |
| 5 | [Management](./machines/management/report.md) | Linux | Easy | OpenAM RCE (CVE-2026-33439, per reference), GLPI credential recovery, LDAP secret decryption, sudo/`rdiff-backup` privilege escalation (per notes) | ✅ Completed |

---

## Statistics

| Stat | Value |
|------|-------|
| 🖥️ Machines Completed | 5 |
| 🐧 Linux Machines | 3 confirmed (Abducted and Principal OS not confirmed here) |
| 🪟 Windows Machines | 0 confirmed |
| ⚡ Easy | 2 |
| 🟡 Medium | 0 confirmed |
| 🔴 Hard | 0 confirmed |
| ❔ Difficulty not specified | 3 |

---

## Attack Techniques Covered

### Web Application
- JavaScript reverse engineering / unpacking
- API enumeration via self-documenting endpoints
- Mass assignment / privilege escalation via API
- Command injection (blind, time-based)
- Base64 / ROT13 encoding/decoding
- OpenAM remote code execution (as described in the Management reference)
- GLPI configuration and database credential discovery
- Decryption of stored LDAP credentials using application key material

### Post Exploitation
- Reverse shell stabilization
- Credential harvesting from configuration files
- Database enumeration (MariaDB/MySQL)
- Password hash cracking (bcrypt)
- Credential reuse and account pivoting (where confirmed by machine notes)

### Privilege Escalation
- Linux kernel exploitation (CVE-2023-0386)
- OverlayFS SUID bit preservation vulnerability
- SUID binary enumeration
- `sudo -l` permission review
- `rdiff-backup` sudo-rule/path-restriction abuse (Management notes; verify against target behavior)
- Symbolic-link abuse / file-handling issues (Abducted notes; see machine report)

---

## Methodology

Every machine follows the same structured approach:

1. Reconnaissance → Nmap and service enumeration
2. Enumeration → web directories, API endpoints, JavaScript analysis, and local enumeration
3. Exploitation → vulnerability identification and validation
4. Post Exploitation → credential discovery and lateral movement
5. Privilege Escalation → local privilege escalation to root where applicable
6. Reporting → structured findings and remediation

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and service enumeration |
| Browser DevTools | JavaScript analysis, API interaction |
| de4js | JavaScript unpacking/beautification |
| Burp Suite | HTTP traffic interception and manipulation |
| Netcat (`nc`) | Reverse shell listener |
| John the Ripper | Password hash cracking |
| hashcat | GPU-accelerated hash cracking |
| GTFOBins | SUID/sudo privilege escalation reference |
| CVE PoC repositories | Known vulnerability research and lab validation |
| MySQL client | Database enumeration |
| PHP / libsodium | Understanding and testing GLPI secret decryption |
| SSH | Remote access and key-based authentication |

---

## Resources & References

- [HackTheBox](https://www.hackthebox.com)
- [GTFOBins](https://gtfobins.github.io/) — Unix binary exploitation reference
- [de4js](https://lelinhtinh.github.io/de4js/) — JavaScript deobfuscator
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — Exploit payloads
- [HackTricks](https://book.hacktricks.xyz/) — Pentesting techniques
- [CVE Details](https://www.cvedetails.com/) — CVE database
- [Management-HTB reference](https://github.com/DIOEK/Management-HTB)
- [OpenAM CVE proof of concept referenced in the write-up](https://github.com/infernosalex/CVE-2026-33439-Python-PoC)

---

## Progress

![Machines Completed](https://img.shields.io/badge/Machines%20Completed-5-brightgreen)
![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?logo=hackthebox)
![Focus](https://img.shields.io/badge/Focus-Linux%20%7C%20Web-blue)

---

*Last updated: October 2026*
