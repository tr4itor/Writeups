**Теги:** Forensics, Windows, Cryptography.
**Уровень**: Hard.
## Описание

В этой комнате необходимо провести форензик-исследование Windows-машины и восстановить цепочку секретов, оставленных на компьютере.

В собранном KAPE-образе находятся:

* реестровые ульи `SAM`, `SYSTEM` и `SECURITY`;
* DPAPI masterkeys пользователя Веры;
* профиль Google Chrome for Testing;
* файлы `Local State` и `Login Data`;
* файл `backup` размером 100 МБ в каталоге документов.

Главная задача — восстановить пароль, который Вера сохранила в браузере, использовать его для открытия зашифрованного контейнера VeraCrypt и найти внутри него флаг.

Ключевая сложность заключается в том, что необходимые данные разбросаны по нескольким уровням защиты Windows.

## Цепочка решения

Весь путь можно представить следующим образом:

```text
SAM + SYSTEM
      ↓
NT-хэш пользователя Vera
      ↓
Пароль Windows: minivera
      ↓
DPAPI masterkey
      ↓
Ключ шифрования Chrome
      ↓
Сохранённый пароль Chrome
      ↓
Пароль VeraCrypt
      ↓
Контейнер backup
      ↓
PDF
      ↓
Изображение внутри PDF
      ↓
Флаг
```

Файл `backup` сразу выглядит подозрительно: его размер составляет ровно `104857600` байт, содержимое имеет высокую энтропию, а стандартный magic header отсутствует.

Это характерно для зашифрованного контейнера. Дополнительная подсказка указывает на версию VeraCrypt `1.26.x`, что подтверждает направление поиска.

---

## 1. Получение NT-хэша Веры

Начнём с ульев Windows Registry.

Переходим в каталог:

```bash
cd KAPE/C/Windows/System32/config
```

Из `SAM` и `SYSTEM` извлекаем локальные учётные записи:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

В выводе находим пользователя `vera`:

```text
vera:1000:...:1241186a4aac4f34f4bf7ace71b396a8:::
```

Полученный NT-хэш:

```text
1241186a4aac4f34f4bf7ace71b396a8
```

Однако одного NT-хэша недостаточно для расшифровки локальных DPAPI masterkeys.

---

## 2. Восстановление пароля Windows

Для локального DPAPI нам понадобится пароль пользователя.

Сохраняем NT-хэш в формате для John the Ripper:

```bash
echo 'vera:$NT$1241186a4aac4f34f4bf7ace71b396a8' > nt.john
```

Запускаем перебор по `rockyou.txt`:

```bash
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john
```

John успешно восстанавливает пароль:

```text
minivera
```

Это пароль учётной записи Windows Веры. Пока это ещё не пароль от контейнера.

---

## 3. Расшифровка DPAPI masterkey

Возвращаемся в корень KAPE-образа:

```bash
cd ../../../../..
```

Находим DPAPI masterkey пользователя:

```text
C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a
```

Задаём путь к masterkey и SID пользователя:

```bash
MK="C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a"
SID="S-1-5-21-2529683458-431225740-1723070931-1000"
```

Теперь используем `impacket-dpapi`:

```bash
impacket-dpapi masterkey -file "$MK" -sid "$SID" -password minivera
```

Masterkey успешно расшифровывается:

```text
Decrypted key: 0x5e5715ec...9d40
```

Полное значение ключа:

```text
5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40
```

---

## 4. Извлечение сохранённого пароля из Chrome

Теперь переходим к Chrome.

Сохранённые учётные данные находятся в SQLite-базе:

```text
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data
```

Копируем её во временный каталог:

```bash
cp "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" /tmp/LoginData
```

Сам ключ шифрования Chrome находится в файле:

```text
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State
```

В нём нас интересует:

```text
os_crypt.encrypted_key
```

Этот ключ дополнительно защищён Windows DPAPI. Поэтому сначала используем уже полученный masterkey.

Для расшифровки можно воспользоваться следующим Python-скриптом:

```python
import json, base64, sqlite3
from impacket.dpapi import DPAPI_BLOB

try:
    from Crypto.Cipher import AES
except ImportError:
    from Cryptodome.Cipher import AES

MK = bytes.fromhex(
    '5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40'
)

ls = json.load(open(
    'C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State'
))

key = DPAPI_BLOB(
    base64.b64decode(ls['os_crypt']['encrypted_key'])[5:]
).decrypt(MK)

for origin, user, pw in sqlite3.connect('/tmp/LoginData').execute(
    'select origin_url, username_value, password_value from logins'
):
    if pw[:3] == b'v10':
        n, ct, tag = pw[3:15], pw[15:-16], pw[-16:]
        print(
            user,
            '=>',
            AES.new(key, AES.MODE_GCM, nonce=n)
               .decrypt_and_verify(ct, tag)
               .decode()
        )
```

Получаем:

```text
VeraSecretVault => Wh4t1sV3raD0inG0nTh1sH0st
```

Таким образом, мы наконец получили пароль от следующего этапа — контейнера VeraCrypt.

---

## 5. Открытие контейнера VeraCrypt

Файл:

```text
C/Users/vera/Documents/backup
```

является VeraCrypt-контейнером.

Открывать его через GUI необязательно. `cryptsetup` умеет работать с VeraCrypt-совместимыми контейнерами:

```bash
sudo cryptsetup open --type tcrypt --veracrypt "C/Users/vera/Documents/backup" veracnt
```

В качестве passphrase используем:

```text
Wh4t1sV3raD0inG0nTh1sH0st
```

Создаём точку монтирования:

```bash
sudo mkdir -p /mnt/vera
```

Монтируем контейнер в режиме только для чтения:

```bash
sudo mount -o ro /dev/mapper/veracnt /mnt/vera
```

После открытия внутри обнаруживается каталог:

```text
secret_financial_documents/
```

В нём находятся:

```text
transactions_q3.csv
important_invoice_byte_lotus.pdf
```

---

## 6. Поиск флага в PDF

Обычный `pdftotext` здесь не поможет, поскольку PDF является изображением, а не обычным текстовым документом.

Поэтому извлекаем встроенные изображения:

```bash
pdfimages -all "/mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf" /tmp/img
```

После этого открываем полученное изображение:

```text
/tmp/img-000.png
```

На изображении находится нужная строка с флагом.

## Флаг

```text
THM{1t_w4s_V3r4_A11_Al0ng?!}
```

---

## Полный набор команд

Для удобства весь процесс:

```bash
cd KAPE/C/Windows/System32/config

# Получаем NT-хэш Vera
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Восстанавливаем пароль Windows
echo 'vera:$NT$1241186a4aac4f34f4bf7ace71b396a8' > nt.john
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john

cd ../../../../..

MK="C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a"
SID="S-1-5-21-2529683458-431225740-1723070931-1000"

# Расшифровываем DPAPI masterkey
impacket-dpapi masterkey -file "$MK" -sid "$SID" -password minivera

# Копируем базу Chrome
cp "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" /tmp/LoginData

# После получения Chrome AES key расшифровываем Login Data
# → VeraSecretVault => Wh4t1sV3raD0inG0nTh1sH0st

# Открываем VeraCrypt-контейнер
sudo cryptsetup open --type tcrypt --veracrypt \
    "C/Users/vera/Documents/backup" veracnt

sudo mkdir -p /mnt/vera
sudo mount -o ro /dev/mapper/veracnt /mnt/vera

# Извлекаем изображения из PDF
pdfimages -all \
    "/mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf" \
    /tmp/img

# Флаг находится в /tmp/img-000.png
# THM{1t_w4s_V3r4_A11_Al0ng?!}

# Очистка
sudo umount /mnt/vera
sudo cryptsetup close veracnt
```

## Что полезно запомнить

* DPAPI-артефакты пользователя можно исследовать офлайн, если доступны masterkeys и необходимые данные для их расшифровки.
* В Chrome цепочка в данном случае выглядит как `Local State → DPAPI → AES key → Login Data → AES-GCM`.
* Файл без заголовка, с размером, кратным типичному размеру контейнера и высокой энтропией, стоит проверить на наличие зашифрованного контейнера.
* Если `pdftotext` ничего не выводит, это ещё не означает, что PDF пустой: содержимое может находиться в виде встроенных изображений.
* При работе с флагами стоит внимательно проверять leetspeak: `V3r4`, `Al0ng` и другие символы могут иметь значение.