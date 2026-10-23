HTB Management — Report
Platform: Hack The Box
Target: Management
Operating System: Linux (Ubuntu)
Difficulty: Easy

Attack Chain Summary
1. Enumerate exposed services with Nmap.
2. Browse the web service and discover the sso.management.htb login portal.
3. Identify OpenAM and use the referenced proof of concept for CVE-2026-33439 to obtain a shell as openam in the lab.
4. Read GLPI configuration files and recover the database connection credentials.
5. Query the GLPI database for the encrypted LDAP bind password.
6. Use GLPI's encryption key and the application's decryption implementation to recover the stored secret.
7. Pivot to owen using the recovered credential, as described in the additional notes.
8. Abuse the permissive rdiff-backup sudo rule to access root's SSH private key.
9. Authenticate as root using the recovered key.
1. Reconnaissance
The supplied Nmap report identified these services:
Port	Service	Notes
22/tcp	SSH	OpenSSH on Ubuntu
80/tcp	HTTP	Redirects to HTTPS
443/tcp	HTTPS	Web application; certificate names management.htb and *.management.htb
1689/tcp	Java RMI	Java remote management interface
4444/tcp	SSL/LDAP-like service	LDAP-related response observed
37339/tcp	Java RMI	Java RMI service
50389/tcp	LDAP	Anonymous bind accepted according to Nmap


Example enumeration command:
nmap -sC -sV -oN nmap_report.txt <TARGET_IP>
The web portal redirected to management.htb. Clicking Client Login led to sso.management.htb.

Add the target hostnames to /etc/hosts using the IP address assigned to the current lab instance:
<TARGET_IP> management.htb sso.management.htb
2. OpenAM Foothold
The reference reports that Burp Suite traffic exposed an OpenAM version string identifying version 16.0.5. The author then referenced a proof of concept for CVE-2026-33439 targeting a password-reset endpoint:
https://sso.management.htb/openam/ui/PWResetUserValidation
The described issue involves a serialized Java object being processed by a vulnerable JATO ViewBean endpoint, potentially leading to remote command execution. In the lab, the author used the referenced proof of concept to obtain a reverse shell as openam.

Evidence to retain: the observed version, the affected endpoint, the request/response, and proof of the resulting shell. Verify the CVE and exploit applicability against the actual lab target before treating the version string alone as proof of vulnerability.
3. GLPI Configuration Discovery
After obtaining the openam shell, the reference describes discovering a local GLPI installation and reading its configuration directory.

Two files were important:
- config_db.php — database connection configuration.
- glpicrypt.key — key used by GLPI to protect sensitive configuration values.
The database configuration contained credentials for the local GLPI database (glpidb). These credentials allowed the database to be queried locally.

Root cause: application configuration and cryptographic key material were readable from the compromised account.
4. Retrieve the Encrypted LDAP Bind Password
The GLPI database contained a table named glpi_authldaps. Its rootdn_passwd field stored the encrypted bind password for the service account:
cn=svc-glpi,ou=services,dc=management,dc=htb
The reference queried the table instead of dumping the entire database:
SELECT * FROM glpi_authldaps;
The retrieved password field looked like Base64, but Base64 is only an encoding. Decoding it reveals the binary encrypted payload, not the plaintext password.
5. Decrypt the Stored Secret
The reference inspected GLPI's own source file, src/GLPIKey.php, to determine the precise cryptographic function and data format. It reports use of libsodium's XChaCha20-Poly1305 authenticated-encryption function, with a 24-byte nonce and the local key file.

The important concepts are:
- The encrypted value contains a nonce followed by ciphertext.
- The key must be read in the exact format expected by GLPI.
- The application’s own implementation is the best reference for nonce handling and decryption parameters.
Root cause: the compromised account could read both the encrypted credential and the key needed to decrypt it.
6. Pivot to owen
The additional notes describe using the recovered directory credential to obtain access as owen. The supplied GitHub excerpt ends before showing the exact authentication command or proving which service accepted the credential, so that transition should be confirmed with lab evidence.

Useful evidence for a final report includes the successful authentication method, the resulting username, and a redacted command output proving the account context.
7. Privilege Escalation via rdiff-backup
The additional notes show this sudo -l entry for owen:
(root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
The intended security controls are:
- Run the backup server as root.
- Restrict its permitted path to /opt/backup.
- Use read-only mode.
The trailing wildcard permits additional arguments matching the sudoers rule. The notes describe appending a second --restrict-path option to override the intended directory restriction. If the installed rdiff-backup version accepts the repeated option with last-value-wins behavior, the restriction can be changed to /, potentially exposing files outside /opt/backup.

The reported impact is access to files under:
/root/.ssh/
The additional notes say the root SSH private key was restored to a temporary directory and then used to authenticate as root.

Root cause: an overly broad sudoers rule allowed caller-controlled arguments to affect a security-sensitive option.

Verification note: confirm the exact command and option-parsing behavior against the target's installed rdiff-backup version and the actual lab output. This section is based on the additional notes, not the supplied GitHub excerpt.
8. Root Access
The additional notes report that the recovered private key was used to connect as root over SSH. After obtaining the root shell, the usual HTB proof file is:
cat /root/root.txt
Do not include actual flag values, private keys, or reusable credentials in a public repository.
Findings and Remediation
Finding	Impact	Recommended remediation
OpenAM remote command execution (as reported by the reference)	May permit command execution as the application account	Patch to a supported fixed release; restrict access to affected endpoints; review application logs
GLPI configuration readable by a non-application user	Database credentials and encryption key exposed	Restrict configuration-directory permissions to the required service account
Encrypted LDAP secret and decryption key accessible together	Stored LDAP credential can be recovered after local compromise	Protect key material separately where supported; rotate exposed service credentials
Broad rdiff-backup sudo rule with trailing wildcard	May allow path-restriction bypass and root-level file access	Replace with a fixed-argument wrapper or narrowly constrained sudo rule; reject conflicting/repeated restriction flags
Potential credential reuse	One recovered secret may enable access to other accounts/services	Use unique credentials and rotate any exposed or reused passwords


Key Lessons
- Enumerate every exposed service, not only HTTP and SSH.
- Treat readable configuration files as potential sources of credentials and cryptographic keys.
- Base64 encoding is not encryption.
- Inspect application source code when you need to understand how a stored secret is encrypted.
- After gaining a foothold, enumerate local services, files, users, and permissions.
- Review sudo -l carefully: wildcard arguments can make a seemingly restricted command unsafe.
- Separate confirmed observations from assumptions in a professional penetration-test report.
References
- Supplied reference repository: https://github.com/DIOEK/Management-HTB
- OpenAM proof of concept cited by the reference: https://github.com/infernosalex/CVE-2026-33439-Python-PoC
