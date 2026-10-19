# Abducted --- Hack The Box

  -----------------------------------------------------------------------
  Field                               Details
  ----------------------------------- -----------------------------------
  Platform                            Hack The Box

  Machine                             Abducted

  Operating System                    Linux

  Difficulty                          Medium

  Status                              Completed

  Primary focus                       Samba enumeration, command
                                      injection, credential discovery,
                                      lateral movement, systemd privilege
                                      escalation
  -----------------------------------------------------------------------

> **Scope:** This report summarizes the attack path described in the
> walkthrough for the authorized Hack The Box lab. Host addresses,
> credentials, and flag values are intentionally omitted. Use only
> against systems you are authorized to test.

## 1. Executive Summary

Abducted is a Linux machine demonstrating how several configuration
weaknesses can be chained to obtain root access. Initial access is
achieved through a vulnerable Samba printing configuration exposed
through a guest-accessible printer share. Post-exploitation enumeration
reveals an rclone configuration containing a reversibly obscured
password. Password reuse provides SSH access as `scott`.

A misconfigured Samba transfer share then permits unsafe symbolic-link
traversal and file operations under a forced user identity, allowing an
SSH public key to be written into `marcus`'s account. Finally, the
`operators` group has write access to a systemd drop-in directory for
the Samba service. This allows a service configuration change that
executes a command with root privileges when the service starts.

### Attack path

``` text
Nmap reconnaissance
    |
    v
SMB enumeration: guest access + writable printer share
    |
    v
Samba print command injection (CVE-2026-4480, as reported by the source)
    |
    v
Initial shell as samba
    |
    v
Discover rclone configuration and recover obscured password
    |
    v
Password reuse -> SSH as scott
    |
    v
Samba force-user + wide-link misconfiguration
    |
    v
Write SSH public key into marcus's account -> SSH as marcus
    |
    v
Writable systemd drop-in directory
    |
    v
Privileged service startup command -> root
```

## 2. Reconnaissance

### Nmap scan

The walkthrough used:

``` bash
nmap <TARGET> -sCV -A -oN abducted-default
```

Key options:

-   `-sC`: run default Nmap scripts.
-   `-sV`: detect service versions.
-   `-A`: enable OS detection, version detection, scripts, and
    traceroute.
-   `-oN`: save output in normal text format.

### Observed services

  -----------------------------------------------------------------------
  Port                    Service                 Relevance
  ----------------------- ----------------------- -----------------------
  22/tcp                  SSH                     Potential remote login
                                                  if valid credentials or
                                                  keys are found

  139/tcp                 NetBIOS/SMB             SMB-related
                                                  communication

  445/tcp                 SMB/Samba               Share enumeration and
                                                  the initial attack
                                                  surface
  -----------------------------------------------------------------------

The scan identified SSH and Samba as the principal services to
investigate. Version detection alone does not prove a vulnerability;
configuration and behavior must also be examined.

## 3. SMB Enumeration

The walkthrough checked whether SMB accepted empty credentials and
enumerated shares:

``` bash
nxc smb <TARGET> -u '' -p '' --shares
```

The results indicated null/guest authentication and a writable share
named `HP-Reception`, described as a reception printer. Other discovered
shares included `projects`, `transfer`, and `IPC$`.

### Why this mattered

-   Empty-credential authentication allowed share enumeration without a
    known account password.
-   The writable printer share provided a lead for investigating Samba's
    printing subsystem.
-   SMB share permissions can expose attack paths that are not apparent
    from the port scan alone.

**Finding:** Guest-accessible SMB resources and a writable printer share
increased the attack surface.

## 4. Initial Access --- Samba Print Command Injection

The source walkthrough identifies **CVE-2026-4480** as a Samba printing
command-injection issue. According to that description, a
client-controlled print-job description is inserted into a configured
print command without correct escaping of shell metacharacters.

Conceptually, the unsafe flow is:

1.  A client submits a print job with a crafted description.
2.  Samba inserts that description into a shell command.
3.  Shell metacharacters are interpreted rather than treated as ordinary
    text.
4.  Commands may execute with the privileges of the affected service.

The walkthrough used the proof of concept published by TheCyberGeek. In
an authorized lab, use the source's documented procedure and substitute
the target, your own VPN-side listener address, listener port, and
printer share.

The reported result was a reverse shell running as the `samba` account.

### Security impact

Remote command execution through a network-accessible printing service
can allow an attacker to execute commands on the server. The resulting
privileges depend on the account and service configuration; initial
service access does not automatically mean root access.

### Evidence to retain

-   Nmap output.
-   SMB share enumeration output.
-   Relevant Samba configuration and version evidence.
-   A timestamped note documenting the authorized proof-of-concept
    result and the account obtained.

## 5. Post-Exploitation --- rclone Configuration

After obtaining the initial shell, the walkthrough searched for
configuration files:

``` bash
find / -type f -name "*.conf" 2>/dev/null | grep -Ev "^/usr/|^/etc/"
```

The search found:

``` text
/opt/offsite-backup/rclone.conf
```

The file contained an entry associated with `svc-backup` and an obscured
password.

### Why this mattered

Rclone is a file synchronization and backup tool. Its configuration can
contain credentials used to access remote storage. Rclone's obscuring
mechanism is reversible and should not be treated as secure encryption.

The walkthrough used `rclone reveal` on the obscured value, then tested
whether the recovered password had been reused by another local account.
The password worked for `scott`, allowing SSH access.

**Finding:** A readable backup configuration exposed a recoverable
secret, and password reuse converted that secret into access to a
separate user account.

> Do not include recovered passwords, tokens, private keys, or other
> secrets in a public write-up.

## 6. User Access --- Scott

After authenticating through SSH as `scott`, the walkthrough checked the
current identity:

``` bash
id
```

The user flag was then retrieved from the account's home directory. The
flag value is deliberately not reproduced here.

This established user-level access, but `scott` did not yet have root
privileges.

## 7. Lateral Movement --- Scott to Marcus

Further enumeration identified a Samba `transfer` share with dangerous
configuration, including:

-   `force user = Marcus`
-   Wide-link behavior enabled in a way that permitted symbolic links to
    point outside the share.

### Why these settings are risky

**Forced user identity:** Samba performs relevant file operations using
the configured Unix identity, subject to the share's permissions.

**Symbolic link:** A symlink is a filesystem object that points to
another path. For example, a link inside a shared directory can point to
a user's home directory.

When forced-user behavior is combined with unsafe link traversal, file
operations through a share may reach locations outside the intended
share boundary. This can undermine the separation between users' files.

### How the walkthrough used the weakness

The walkthrough generated an SSH key pair, created a symbolic link
targeting Marcus's home directory, and used the transfer share to place
the public key in Marcus's `authorized_keys` file. The matching private
key was then used to authenticate as `marcus`.

The key concept is that the attacker did not need to crack Marcus's
password. Instead, the misconfigured share enabled modification of an
authentication file belonging to another user.

### Security impact

An attacker with access to the share may be able to write files outside
the intended share boundary or modify another user's SSH authorization,
depending on the effective permissions and configuration.

## 8. Privilege Escalation --- Marcus to Root

The walkthrough checked Marcus's groups:

``` bash
id
```

Marcus belonged to the `operators` group. A filesystem search
identified:

``` text
/etc/systemd/system/smbd.service.d
```

The walkthrough reported that the `operators` group could write to this
directory.

### Why the directory matters

Systemd drop-in directories allow administrators to extend or override
service-unit settings. Files in a service's drop-in directory can affect
how the service starts.

The Samba service, `smbd.service`, normally runs with elevated
privileges. If an untrusted user can write a service drop-in that
introduces a command executed during service startup, the user may be
able to cause privileged command execution.

The walkthrough added a drop-in containing an `ExecStartPre` command
that changed the SUID bit on the root-owned Bash executable. It then
reloaded systemd's unit definitions and restarted Samba. Finally, Bash
was invoked in privileged mode, and the reported effective user ID was
`0`.

### Root cause

The primary issue was excessive write access to privileged service
configuration. The ability to alter service startup behavior created a
route from the `operators` group to root-level command execution.

### Important operational note

On a real host, whether a user can reload systemd configuration or
restart a service depends on authorization and service-control policy. A
writable drop-in directory is still a serious configuration defect even
if a separate authorization check prevents the exact sequence in a
particular environment.

## 9. Findings and Remediation

  -----------------------------------------------------------------------
  Finding                 Risk                    Recommended remediation
  ----------------------- ----------------------- -----------------------
  Guest/null SMB          Unauthenticated users   Disable guest access
  authentication          may enumerate or access where unnecessary;
                          shares                  enforce least privilege

  Writable printer share  Potential remote        Patch Samba to a
  exposed to vulnerable   command execution       vendor-supported fixed
  print handling                                  version; disable
                                                  unnecessary printing
                                                  functionality

  Readable rclone         Secret disclosure and   Restrict file
  configuration with      account compromise      permissions, rotate
  recoverable credentials                         exposed credentials,
                                                  use a proper
                                                  secrets-management
                                                  solution

  Password reuse between  A recovered secret can  Use unique passwords
  accounts                provide access to       and a password manager;
                          another account         rotate reused
                                                  credentials

  Forced-user share       Possible access beyond  Disable wide links
  combined with unsafe    the intended share      unless strictly
  wide links              boundary                required; review Unix
                                                  extensions and share
                                                  settings; remove
                                                  forced-user designs
                                                  that cross trust
                                                  boundaries

  Writable systemd        Potential root-level    Restrict ownership and
  drop-in directory for a command execution       write permissions to
  non-administrative                              root/trusted
  group                                           administrators; audit
                                                  service unit files and
                                                  drop-ins

  SUID bit on Bash        Privileged shell may be Remove unnecessary SUID
                          obtainable by local     permissions and
                          users                   investigate how they
                                                  were introduced
  -----------------------------------------------------------------------

## 10. Lessons Learned

1.  **Enumerate services and shares systematically.** A writable SMB
    share can provide a more promising lead than SSH.
2.  **Treat service configuration as sensitive.** Printing commands and
    shell substitutions can create command-injection vulnerabilities.
3.  **Search for secrets after gaining a foothold.** Backup
    configuration files and scripts may expose credentials.
4.  **Understand symlinks and effective identities.** A share's
    filesystem behavior can undermine user separation.
5.  **Check group permissions, not just user permissions.** Group
    membership may grant access to sensitive service configuration.
6.  **Inspect privileged execution paths.** Writable systemd units,
    drop-ins, scripts, and service directories can lead to root.
7.  **Collect evidence as you go.** Save scans, relevant configuration
    excerpts, permission output, timestamps, and clear notes about each
    transition.

## 11. Evidence Checklist

-   [ ] Nmap scan saved to the report's evidence folder.
-   [ ] SMB share and permission enumeration captured.
-   [ ] Samba version/configuration evidence recorded.
-   [ ] Initial access documented without publishing sensitive payloads
    or credentials.
-   [ ] Relevant rclone configuration path and file permissions
    recorded; secrets redacted.
-   [ ] Password-reuse finding documented without publishing the
    password.
-   [ ] Samba share configuration and symlink behavior recorded.
-   [ ] `operators` group membership and drop-in directory permissions
    captured.
-   [ ] Root-level impact verified in the authorized lab without
    including flag values.
-   [ ] Remediation recommendations tied to each finding.

## 12. Conclusion

Abducted demonstrates a chained compromise rather than a single isolated
exploit. A Samba printing weakness provides initial access, an exposed
backup configuration reveals a reusable credential, a misconfigured SMB
share enables lateral movement, and writable systemd service
configuration allows privilege escalation to root.

The central lesson is that small configuration mistakes across different
components can combine into a complete system compromise. Effective
defense requires patching exposed services, restricting credentials and
shares, preserving filesystem boundaries, and protecting privileged
service configuration.

------------------------------------------------------------------------

*Write-up prepared for an authorized Hack The Box lab. CVE details and
exploit behavior above reflect the supplied walkthrough; independently
verify vendor advisories and affected-version information before using
the finding in a real-world assessment.*
