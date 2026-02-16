# Penetration Testing Report: RootMe (TryHackMe)

## 1. Overview
This report documents the exploitation process of the **RootMe** machine on TryHackMe. The laboratory focuses on web reconnaissance, file upload bypass, and Linux privilege escalation via SUID binaries.

---

## 2. Reconnaissance (Scanning)
The initial stage involved identifying open ports and services using **Nmap**:
```bash
nmap -sV -sC 10.65.159.181
Port 22 (SSH): Open (OpenSSH 7.6p1)

Port 80 (HTTP): Open (Apache 2.4.29)

Directory Enumeration
To find hidden paths, I used GoBuster:

/panel/ - An upload form for files.

/uploads/ - The directory where uploaded files are stored.

3. Initial Access (Exploitation)
The /panel/ directory was vulnerable to Unrestricted File Upload, though it had a basic extension filter.

Bypass Steps:
I attempted to upload a standard .php shell, which was blocked.

I bypassed the filter by changing the file extension to .phtml.

Started a local listener using Netcat:

Bash
nc -lvnp 1234
Triggered the shell by navigating to: http://10.65.159.181/uploads/shell.phtml

Result: Established a reverse shell as the www-data user.

4. Privilege Escalation
After gaining access, I looked for misconfigured SUID binaries:

Bash
find / -user root -perm -4000 -print 2>/dev/null
Discovery: The binary /usr/bin/python2.7 had the SUID bit set.

Exploitation:
I used the following Python command to elevate my privileges to Root:

Bash
python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
Verification: whoami -> root

5. Captured Flags
User Flag: Found in /var/www/user.txt

Root Flag: Found in /root/root.txt

6. Remediation (Security Tips)
Sanitize Inputs: Use a Whitelist approach for file uploads (allow only .jpg, .png).

Binary Hardening: Remove the SUID bit from powerful interpreters like Python.

Network Security: Implement firewall rules to block suspicious outbound connections (Reverse Shells).
