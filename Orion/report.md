# HTB Orion --- Machine Write-up

> **Platform:** Hack The Box\
> **Target:** Orion\
> **Target IP:** `10.129.244.146`\
> **Hostname:** `orion.htb`\
> **Difficulty:** Verify against the machine listing before publishing\
> **Status:** Completed
>
> This write-up documents the attack path used in the lab. Verify
> commands and findings against your own evidence before publication.
> Keep this content limited to the authorized HTB environment.

## Overview

Orion was approached using an enumeration-first workflow. SSH and HTTP
were identified, the web application was identified as Craft CMS, and an
initial foothold was obtained through a reported application
vulnerability. Configuration data exposed database connection details, a
password hash was recovered, and password reuse enabled SSH access as
`adam`. Local service enumeration then identified a Telnet service bound
to loopback, which provided the reported final privilege-escalation
path.

**High-level chain:**
`Nmap → HTTP enumeration → Craft CMS → initial foothold → exposed .env → password hash recovery / password reuse → SSH as adam → local Telnet service → root`

## 1. Reconnaissance

``` bash
nmap -sC -sV -oN nmap.txt 10.129.244.146
```

The reported scan identified `22/tcp` (SSH) and `80/tcp` (HTTP). Confirm
the target IP is current before reusing it; HTB addresses can change.

If required, map the lab hostname in `/etc/hosts`:

``` text
10.129.244.146  orion.htb
```

## 2. Web Enumeration

Directory enumeration was performed with Gobuster:

``` bash
gobuster dir \
  -u http://orion.htb/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster.txt
```

Reported paths included `/admin/login` and `/assets`. Craft CMS was
identified, with version `5.6.16` reported during the investigation.
Confirm this version from saved evidence before publishing.

The key step was identifying the CMS and researching the security
implications of its installed version, rather than relying on the admin
login page alone.

## 3. Initial Access

The reported lab path used a Craft CMS vulnerability associated with
**CVE-2025-32432** to obtain an initial shell.

Before publishing, verify the CVE, affected-version range, and exploit
conditions against a trusted advisory. A version string alone does not
prove exploitability. Record the exact exploit source, command or
module, target response, and shell identity from your own session. Avoid
including unverified exploit commands.

After obtaining a shell, verify the context:

``` bash
whoami
id
pwd
```

## 4. Credential Discovery and Cracking

An exposed `.env` file reportedly revealed database connection details.
Such files may contain database credentials, application secrets, and
other sensitive configuration.

**Impact:** An attacker who can read application configuration may gain
credentials that can be used to access databases or other services.

Do not publish actual passwords, tokens, session secrets, complete
credential dumps, or recovered flags. Redact secrets in screenshots and
examples.

The lab notes indicate that Hashcat mode `3200` was used for a bcrypt
password hash:

``` bash
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```

Use this mode only after correctly identifying the hash format. The
recovered password was reportedly reused for the system account `adam`,
enabling SSH access.

## 5. SSH Access

``` bash
ssh adam@orion.htb
```

After logging in, enumerate and verify the account:

``` bash
whoami
id
hostname
uname -a
```

The reported result was a user-level foothold as `adam`. Do not include
the user flag value in the public repository.

## 6. Privilege Escalation

Local enumeration identified a Telnet listener bound to loopback:

``` bash
ss -lntp
```

A loopback-only service may not be reachable directly from outside the
machine, but it can still expand the attack surface after a shell is
obtained.

The reported final step was associated with **CVE-2026-24061**, an
authentication-bypass issue involving GNU Inetutils telnetd. Verify the
running implementation, package version, and vulnerability conditions
against a trusted advisory before attributing the result to this CVE.
Reproduce the technique only within the authorized HTB environment.

After testing the lab path, verify any privilege transition:

``` bash
whoami
id
```

The final state was reported as root. Do not publish the root flag or
sensitive machine data.

## Key Findings

### 1. Vulnerable web application --- Initial access

-   **Impact:** A web application vulnerability reportedly enabled an
    initial foothold under the lab's conditions.
-   **Evidence:** CMS/version evidence, relevant request and response,
    exploit output, and shell identity.
-   **Remediation:** Upgrade to a fixed supported version, restrict
    administrative access, and review application logs.

### 2. Exposed application configuration

-   **Impact:** Configuration files may disclose database credentials
    and application secrets.
-   **Evidence:** Redacted file path and proof of access; do not publish
    secret values.
-   **Remediation:** Prevent web access to configuration files, apply
    least-privilege permissions, store secrets securely, and rotate
    exposed credentials.

### 3. Password hash recovery and password reuse

-   **Impact:** A recoverable hash and reused password reportedly
    enabled access to another service.
-   **Evidence:** Hash-format identification and redacted cracking
    output; do not publish plaintext passwords.
-   **Remediation:** Enforce unique passwords, MFA where available, and
    an appropriate password-hashing scheme with a suitable work factor.

### 4. Vulnerable local Telnet service

-   **Impact:** A local service with a reported authentication-bypass
    weakness enabled privilege escalation in the lab.
-   **Evidence:** Listener details, service/package version, trusted
    advisory, and before/after identity output.
-   **Remediation:** Patch or remove the vulnerable service, disable
    unnecessary Telnet, prefer SSH, and restrict service access.

## Lessons Learned

-   Enumerate services before choosing an exploitation path.
-   Identify the web framework or CMS and verify its version with
    evidence.
-   Treat configuration files as sensitive assets.
-   Check whether recovered credentials are reused across services.
-   Inspect local listening services, including loopback-only listeners.
-   Verify service implementation and version before assigning a CVE.
-   Confirm privilege transitions using `whoami` and `id`.
-   Preserve evidence throughout the assessment.

## Remediation Summary

1.  Patch the vulnerable CMS and affected local service.
2.  Remove public access to `.env` and other sensitive configuration
    files.
3.  Rotate exposed database credentials, application secrets, and reused
    passwords.
4.  Enforce unique credentials and MFA where supported.
5.  Disable unnecessary Telnet and prefer SSH.
6.  Apply least privilege to application, database, and operating-system
    accounts.
7.  Review relevant web, authentication, and service logs.
8.  Add regression tests and configuration checks to prevent recurrence.

## Evidence Checklist

-   [ ] Nmap output showing ports and services
-   [ ] Web enumeration output
-   [ ] CMS and version evidence
-   [ ] Trusted advisory and exact conditions for the initial-access
    vulnerability
-   [ ] Redacted proof of configuration exposure
-   [ ] Hash type identification and redacted Hashcat output
-   [ ] SSH login and `whoami`/`id` output as `adam`
-   [ ] Telnet listener and service/package version
-   [ ] Trusted advisory confirming the Telnet vulnerability
-   [ ] `whoami`/`id` output verifying root access
-   [ ] Passwords, hashes, tokens, flags, and personal details removed
    from public artifacts

## References

-   Hack The Box: https://www.hackthebox.com/
-   Nmap reference guide: https://nmap.org/book/man.html
-   Gobuster: https://github.com/OJ/gobuster
-   Hashcat documentation: https://hashcat.net/wiki/
-   Craft CMS knowledge base: https://craftcms.com/knowledge-base
-   NVD CVE search: https://nvd.nist.gov/vuln/search

> **Publication note:** Confirm the machine difficulty, exact CMS
> version, affected service version, and CVE details against your own
> evidence and trusted advisories. This report intentionally omits flags
> and recovered secrets.
