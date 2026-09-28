# TryHackMe — Startup SpiceHut
IP: 10.11.111.15 Total time spent in the room: Approximately 100 minutes.

We are tasked with finding a so-called "secret recipe", as well as the `user.txt` and `root.txt` files.

## 1. Reconnaissance

First, we check whether a website is running on the target's IP address. It is.

The homepage doesn't reveal anything interesting. The page source also contains nothing useful.

Let's move on to scanning tools.

### Nmap

We use Nmap with the `-sS` and `-sV` flags to identify open ports and the services running on them.


```BASH
sudo nmap -sS -sV 10.11.111.15
```

The scan returns the following results:

```
Nmap scan report for 10.11.111.15

Starting Nmap 7.92 (https://nmap.org) at 2022-09-28 14:15 EDT
Host is up (0.081s latency).
Not shown: 997 closed tcp ports (conn-refused)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.18
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

We immediately notice three open ports:

* 21/tcp — FTP

* 22/tcp — SSH

* 80/tcp — HTTP

I initially searched vulnerability databases for exploits affecting these service versions. I found several potential exploits, but none of them were suitable for this target.

## 2. FTP Enumeration

We try connecting to the FTP server using anonymous authentication.

After successfully logging in, we find the following files and directories:

```
notice.txt
important.jpg
.test.log
ftp/
```

I inspect the available files. The `.test.log` file contains only the word `test`, while the other files don't reveal anything particularly interesting.

Before exploring the directory, I check the available FTP commands using:

```
?
```

The available commands include `get` and `put`.

I try using `put` to upload a file, but initially, the attempt is unsuccessful.

Using `get`, I download and inspect the available files, including `.test.log`, `notice.txt`, and `important.jpg`.

At this point, I think I've exhausted the available information and decide to explore the `ftp` directory.

Inside it, I try using `put` again.

This time, it works!

Apparently, this directory allows us to upload files.

## 3. Directory Enumeration

The next day, I remember that I haven't properly enumerated the website's directories.

I decide to run Gobuster using a standard wordlist.


```BASH
gobuster dir -u http://10.11.111.15 -w ~/Downloads/common.txt
```

Gobuster discovers several paths with HTTP status codes `403`, `301`, and `200`.

The most interesting result is:

```
/files
```

It returns status code `301`, indicating a redirect.

When I visit `/files`, I find the same files and directories that were available through FTP, except for the hidden `.test.log` file.

This confirms that files uploaded through FTP may also be accessible through the web server.

## 4. Obtaining a Reverse Shell

At this point, I try to find vulnerabilities in `vsftpd`, but without success.

It becomes clear that I need to look for another way to exploit the system, rather than focusing exclusively on the FTP service.

While researching, I come across a PHP web shell that allows commands to be executed on Linux and Windows systems.

The web shell itself doesn't exploit a vulnerability. The actual weakness is that we can upload files through FTP and that PHP files are executed by the web server.

This means we can upload a PHP reverse shell and execute it through the browser.

First, we start a Netcat listener on our attacking machine:


```BASH
nc -lvnp 4444
```

Next, we upload the PHP reverse shell to the FTP directory.

Once the file has been uploaded, we access it through the browser.

When the reverse shell starts executing, Netcat catches the incoming connection, giving us access to the target's shell.

We can now execute commands. For example:

```BASH
ls -la
```

This allows us to inspect the files and directories accessible to our current user.

However, our access is still limited.

## 5. Finding the Secret Recipe

With access to the PHP shell, we begin searching for the secret recipe and the two flag files.

During enumeration, we discover an interesting directory:

```
incidents/
```

We also find a file named:

```
recipe.txt
```

The file reveals the secret ingredient:

# Love

The secret ingredient

The `incidents` directory contains a network capture file (`.pcapng`).

We can analyze it using Wireshark.

## 6. Analyzing the Network Capture

We download the `.pcapng` file and open it in Wireshark.

While examining the captured traffic, we discover login attempts and successful authentication attempts by different users.

By inspecting the packets, we find the password belonging to the user Lenny.

Now that we have valid credentials, we can connect to the machine over SSH.


```BASH
ssh lenny@10.11.111.15
```

We enter the password we discovered in the network capture.

After successfully logging in, we gain access to Lenny's home directory.

There, we find the `user.txt` file.

User flag

`THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`

## 7. Privilege Escalation

While exploring the system as Lenny, we discover a directory named:

```
/script
```

It belongs to root and contains a script called:

```
planner.sh
```

After inspecting `planner.sh`, we notice that it executes another script located in a different directory.

That second script belongs to our user and contains the following command:

```
Done!
```

This is an interesting discovery because the root-owned script executes a script that we can modify.

We also discover that the script is executed every minute by a cron job.

This gives us an opportunity to escalate our privileges.

### Exploiting the Cron Job

Since we have write access to the script executed by `planner.sh`, we can modify it and insert a command to launch a reverse shell.

First, we replace the contents of our script with a reverse shell command configured to connect back to our attacking machine.

On the attacking machine, we start another Netcat listener:


```BASH
nc -lvnp 4444
```

Once the cron job executes our modified script, the reverse shell connects to our listener.

Because the script is executed by the root-owned process, we obtain a shell with root privileges.

We can verify this by running:


```
id
```

We now have root access and can read the final flag.


```
cat /root/root.txt
```

Root flag

`THM{f963aaa6a430f210222158ae15c3d76d}`


## 8. Exploitation Chain

1. Nmap scan to identify open ports and services.

2. FTP enumeration using anonymous authentication.

3. Discovery of an FTP directory that allows file uploads.

4. Gobuster directory enumeration.

5. Uploading a PHP reverse shell through FTP.

6. Obtaining an initial shell through the web server.

7. Discovering the `recipe.txt` file and the secret ingredient.

8. Finding a network capture in the `incidents` directory.

9. Analyzing the capture in Wireshark to recover Lenny's password.

10. Connecting through SSH as `lenny`.

11. Retrieving `user.txt`.

12. Discovering the root-owned `planner.sh` script and its cron job.

13. Modifying a script executed by the cron job.

14. Obtaining a root shell through a reverse shell.

15. Retrieving `root.txt`.

## 10. Useful Resources

- [Hackviser — FTP Pentesting](https://hackviser.com/tactics/pentesting/services/ftp)
