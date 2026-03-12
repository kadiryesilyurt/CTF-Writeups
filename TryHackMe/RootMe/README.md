# TryHackMe CTF Write-up: RootMe

**Difficulty:** Easy  
**Objective:** Gain initial access (User Flag) and escalate privileges to root (Root Flag).

## 1. Reconnaissance

The first step of any penetration test is understanding the target's architecture. I started by scanning the target's external ports and running services using **Nmap**.

* **Command:** `nmap -sC -sV <Target_IP>`
    * `-sC`: Runs default enumeration scripts.
    * `-sV`: Determines the exact service versions running on the open ports.

**Findings:**
* **Open Ports:** Port 22 and Port 80 are open.
* **Port 22:** Running SSH.
* **Port 80:** Running HTTP (Apache Web Server version 2.4.29).

Knowing there is a web server running on Port 80, I proceeded to perform directory bruteforcing using **Gobuster** to uncover any hidden endpoints that developers might have left exposed.

* **Command:** `gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt`
* **Hidden Directories Found:** The scan revealed two critical directories: `/uploads` and `/panel`.

## 2. Getting a Shell (Foothold)

Navigating to `http://<Target_IP>/panel` in the browser revealed an exposed File Upload form.

**Vulnerability Analysis & Exploitation:**
Often, developers forget to properly validate or filter the extensions of uploaded files. If an application accepts any file type instead of restricting it to safe formats (like `.jpg` or `.png`), it opens the door for **Remote Code Execution (RCE)** by uploading a malicious script written in the server's backend language (PHP in this case).

1.  **Payload Preparation:** I utilized the standard `php-reverse-shell.php` script (by Pentestmonkey) and configured the `$ip` and `$port` variables to point back to my VPN IP address and a listening port (e.g., 4444).
2.  **Filter Bypass:** Upon attempting to upload the `.php` file, the application blocked it, indicating a weak "Blacklist" filtering mechanism. I bypassed this filter by renaming my payload to `shell.phtml`. The Apache server was misconfigured to interpret `.phtml` as valid PHP code, allowing the upload to succeed.
3.  **Catching the Shell:** I set up a Netcat listener on my local machine (`nc -lvnp 4444`).
4.  **Execution:** By navigating to the `/uploads` directory in the browser and clicking on `shell.phtml`, the script executed on the server, granting me a reverse shell as the `www-data` user.

**User Flag:**
To quickly locate the user flag within the filesystem, I used the `find` command, redirecting errors to `/dev/null` for a cleaner output:
* `find / -type f -name user.txt 2>/dev/null`
* **Result:** `THM{[REDACTED]}`

## 3. Privilege Escalation

Having gained initial access as a low-privileged user (`www-data`), the next objective was to escalate privileges to `root`. I started searching for files with **SUID (Set User ID)** permissions that were owned by the root user.

**Vulnerability Analysis:**
When a binary has the SUID bit set, it executes with the privileges of the file's owner (usually root), regardless of who runs it. If a standard tool capable of executing system commands (like Python, Bash, or Nmap) has this permission, it can be abused to spawn a root shell.

**Enumeration:**
* **Command:** `find / -user root -perm -4000 -type f 2>/dev/null`
* **Finding:** The scan revealed that `/usr/bin/python` surprisingly had the SUID bit set. This is a massive security misconfiguration.

**Exploitation via GTFOBins:**
I consulted **GTFOBins**, an industry-standard repository for bypassing local security restrictions using misconfigured binaries. Since Python was available with SUID permissions, I used the `os` library to spawn a shell. 

By passing the `-p` parameter to `/bin/sh`, I prevented the shell from dropping its elevated privileges:
* **Command Executed:** `/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'`

This command successfully bypassed the drop-privilege mechanism and provided a fully interactive `root` shell.

**Root Flag:**
* Executed `cat /root/root.txt` to retrieve the final flag.
* **Result:** `THM{[REDACTED]}`