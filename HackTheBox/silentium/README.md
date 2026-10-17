# 🛡️ HackTheBox: Silentium - Write-up

* **Platform:** HackTheBox
* **Machine Name:** Silentium
* **Difficulty:** Easy / Medium
* **Key Vulnerabilities:** Information Disclosure, CVE-2025-59528 (RCE), Docker Env Credential Leak, Gogs Symlink Arbitrary File Write (CVE-2025-8110)

---

## 🎯 Executive Summary
In this machine, we start by enumerating a hidden virtual host (`staging.silentium.htb`) where an Information Disclosure vulnerability in the authentication and password reset API leaks a valid email and reset token. Using this, we gain administrative access and exploit **CVE-2025-59528** to achieve Remote Code Execution (RCE) inside a minimal Docker container. After extracting SSH credentials from the container's environment variables, we pivot to the host user, port-forward an internal **Gogs** service running as `root`, and exploit an arbitrary symlink vulnerability (**CVE-2025-8110**) to inject our SSH public key into `/root/.ssh/authorized_keys`.

---

## 🔍 1. Reconnaissance & Subdomain Enumeration

### Port Scanning
We start with an `nmap` scan to identify open ports and running services:

```bash
nmap -sC -sV -oN nmap_scan.txt <TARGET_IP>

The scan reveals two open ports:

    Port 22: OpenSSH

    Port 80: HTTP Web Server

We add silentium.htb to our /etc/hosts file:
  echo "<TARGET_IP> silentium.htb" | sudo tee -a /etc/hosts

Directory & Subdomain Fuzzing

Navigating to http://silentium.htb shows a blank/default page, and directory brute-forcing with gobuster yields nothing useful.

Next, we fuzz for virtual hosts (subdomains) using ffuf:
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u [http://silentium.htb](http://silentium.htb) -H "Host: FUZZ.silentium.htb" -fs <SIZE>

We successfully discover a new subdomain: staging.silentium.htb. After adding it to /etc/hosts, we navigate to it in our browser and find a login portal.

🔓 2. Authentication Bypass via Information Disclosure

    Email Enumeration:
    We attempt a login with random credentials on staging.silentium.htb and intercept the request using Burp Suite. Inspecting the HTTP response reveals an Information Disclosure flaw that indicates whether an email address is valid. By testing usernames found on the main page, we identify a valid administrator email.

    Password Reset Token Leak:
    We navigate to the Forgot Password page, submit the valid email, and intercept the request in Burp Suite again. Looking closely at the server's HTTP response, we notice another information disclosure leaking both a password hash and the Password Reset Token directly in the response body!

    Account Takeover:
    Instead of wasting time cracking the hash, we use the leaked reset token along with the valid email on the password reset page, change the password, and log in with Admin privileges.

💥 3. Initial Foothold & Docker Escape (CVE-2025-59528)
Exploiting CVE-2025-59528 (RCE)

After logging into the dashboard, we identify the specific software version running on the server. Searching for public exploits reveals that this version is vulnerable to CVE-2025-59528 (Remote Code Execution).
Overcoming the Missing /bin/bash Limitation

While attempting to trigger a standard reverse shell, the connection fails because the application is running inside a minimal Docker container that does not have bash installed.

To bypass this, we craft a custom Python script / .sh payload using standard /bin/sh to establish a reverse shell back to our Kali machine.
Harvesting Credentials from Environment Variables

Once inside the container, we notice we have root privileges within the Docker environment, but we are isolated from the main host. Enumerating the system configuration and Environment Variables (env) reveals hardcoded SSH credentials for a user on the host machine:
env

We use these credentials to log into the actual host machine via SSH:
ssh <username>@silentium.htb

🚀 4. Privilege Escalation (CVE-2025-8110 in Gogs)
Internal Service Enumeration

Now that we have a foothold on the host via SSH, we enumerate running processes to find privilege escalation vectors:
ps aux | grep root

We discover an internal Gogs (Git service) instance running locally with root privileges.
Port Forwarding

To access the internal Gogs web interface from our Kali machine, we set up SSH Local Port Forwarding:
   ssh -L 3000:127.0.0.1:3000 <username>@silentium.htb


Exploiting Gogs Symlink Vulnerability (CVE-2025-8110)

We open http://127.0.0.1:3000 in our browser, register a new account, and check the Gogs version in the footer. Researching this version reveals CVE-2025-8110, a vulnerability where Gogs fails to properly validate symbolic links (symlinks) inside Git repositories, allowing arbitrary file overwrite.

Since Gogs is running as root, we can abuse this to write our own SSH public key into /root/.ssh/authorized_keys:

   1: Generate an SSH Keypair on Kali:
ssh-keygen -t rsa -b 4096 -f root_key

2: Create a Repository & Inject the Symlink:
We clone our newly created Gogs repository to our Kali machine, create a symlink pointing to the root user's authorized_keys file, and push it back:
ln -s /root/.ssh/authorized_keys .
git add .
git commit -m "Symlink exploit"
git push origin main

3: Trigger the Overwrite via API:
Using curl with our Gogs API authentication token, we update/overwrite the symlink target with our generated SSH public key (root_key.pub).

4: Root Shell:
Finally, we connect to the server as root using our private key:
ssh -i root_key root@silentium.htb

Boom! We are root! 🎉































