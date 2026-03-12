# TryHackMe CTF Write-up: RootMe

**Difficulty:** Easy  
**Platform:** TryHackMe  
**Focus Areas:** Web Enumeration, Bypassing File Upload Filters, SUID Privilege Escalation.

## 1. Reconnaissance & Enumeration

The first phase of the penetration test involved understanding the target's external attack surface. I utilized **Nmap** to discover open ports, running services, and their specific versions.

* **Command:** `nmap -sC -sV -O <Target_IP>`
    * `-sC`: Executes a set of standard NSE (Nmap Scripting Engine) scripts for basic vulnerability checking.
    * `-sV`: Probes open ports to determine the exact service and version info, which is crucial for finding specific exploits.

**Initial Findings:**
* **Port 22 (SSH):** Open, running standard SSH services.
* **Port 80 (HTTP):** Open, running Apache Web Server (version 2.4.29).

Knowing that a web application was hosted on Port 80, I proceeded to map out its structure. Developers often rely on "Security through Obscurity" by simply hiding sensitive directories without implementing proper access controls. To uncover these, I performed a directory bruteforce attack using **Gobuster**.

* **Command:** `gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt`
* **Discovery:** The scan successfully revealed two critical endpoints:
    * `/uploads`: A directory likely used for storing user-uploaded files.
    * `/panel`: A hidden administrative or user interface containing a file upload form.

## 2. Gaining a Foothold (Initial Access)

Navigating to `http://<Target_IP>/panel` presented a file upload mechanism. In web application architecture, failing to properly validate and restrict uploaded file types can lead to severe security breaches, specifically **Remote Code Execution (RCE)**.

**Exploitation Methodology:**
1.  **Weaponization:** I prepared a standard PHP Reverse Shell payload (courtesy of Pentestmonkey). I modified the `$ip` and `$port` variables within the script to establish a reverse TCP connection back to my local attacking machine (Kali Linux).
2.  **Filter Evasion (Blacklist Bypass):** My initial attempt to upload a standard `.php` file was blocked by the application with a "PHP não é permitido!" (PHP is not allowed) error. This indicated the presence of a weak, blacklist-based validation mechanism.
    * *The Bypass:* Instead of a robust whitelist, the developer only blocked the `.php` extension. I renamed my payload to `shell.phtml`. The backend Apache server was misconfigured to pass `.phtml` files to the Zend Engine, which interpreted and executed it as valid PHP code. The upload was successful.
3.  **Execution:** I set up a Netcat listener on my terminal (`nc -lvnp 4444`). Then, I navigated to `http://<Target_IP>/uploads/shell.phtml` via the browser. By sending an HTTP GET request to this file, the server executed the script, forcing it to initiate a connection back to my machine.
4.  **Access:** I successfully caught the reverse shell, gaining interactive access to the system as the `www-data` service user.

**User Flag Retrieval:**
To locate the user flag efficiently across the entire filesystem, I utilized the Linux `find` command, redirecting standard error (`stderr`) to `/dev/null` to filter out "Permission Denied" clutter:
* `find / -type f -name user.txt 2>/dev/null`
* **Result:** `THM{[REDACTED]}`

## 3. Privilege Escalation

With a low-privileged shell (`www-data`), the ultimate objective was to vertically escalate privileges to the `root` user. A common architectural misconfiguration in Linux environments is the improper assignment of **SUID (Set Owner User ID)** permissions.

**Vulnerability Analysis:**
When a binary executes with the SUID bit set, it runs with the privileges of the file's owner rather than the user executing it. If a binary owned by `root` allows arbitrary command execution (such as interpreters like Python or Bash), it can be trivially exploited to spawn a root shell.

**Enumeration:**
I searched the filesystem for binaries owned by root that had the SUID bit (4000) enabled:
* **Command:** `find / -user root -perm -4000 -type f 2>/dev/null`
* **Finding:** The output revealed `/usr/bin/python` with SUID permissions. This is a critical misconfiguration, as Python can easily interact with the underlying operating system.

**Exploitation via GTFOBins:**
Following industry best practices, I referenced **GTFOBins** to find the standard one-liner for exploiting an SUID-enabled Python binary. I used the `os` module to execute a system shell. 

Crucially, modern shells like `/bin/sh` often drop elevated privileges automatically when executed. To prevent this security mechanism, I passed the `-p` (privileged) flag:
* **Command Executed:** `/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'`

This command successfully replaced the current Python process with a new shell process that retained the effective user ID of `root`.

**Root Flag Retrieval:**
* Executed `whoami` to confirm root privileges.
* Executed `cat /root/root.txt` to retrieve the final flag.
* **Result:** `THM{[REDACTED]}`
