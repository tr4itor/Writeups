**Tags:** Forensics, Windows, Cryptography.

**Difficulty:** Hard.

## Description

In this room, we need to perform a forensic investigation of a Windows machine and recover a chain of secrets left on the computer.

The collected KAPE image contains:

* Windows Registry hives `SAM`, `SYSTEM`, and `SECURITY`;
* DPAPI masterkeys belonging to Vera;
* a Google Chrome for Testing profile;
* `Local State` and `Login Data` files;
* a 100 MB `backup` file in the Documents directory.

The main task is to recover the password Vera saved in the browser, use it to open an encrypted VeraCrypt container, and find the flag inside.

The key difficulty is that the required data is spread across several layers of Windows protection.

## Solution Chain

The entire path can be represented as follows:

```text
SAM + SYSTEM
      ↓
Vera's NT hash
      ↓
Windows password: minivera
      ↓
DPAPI masterkey
      ↓
Chrome encryption key
      ↓
Saved Chrome password
      ↓
VeraCrypt password
      ↓
backup container
      ↓
PDF
      ↓
Image inside PDF
      ↓
Flag
```

The `backup` file immediately looks suspicious: its size is exactly `104857600` bytes, its contents have high entropy, and the standard magic header is missing.

This is characteristic of an encrypted container. An additional hint points to VeraCrypt version `1.26.x`, which confirms this direction.

---

## 1. Obtaining Vera's NT Hash

Let's start with the Windows Registry hives.

Navigate to the directory:

```bash
cd KAPE/C/Windows/System32/config
```

Extract the local accounts from `SAM` and `SYSTEM`:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

In the output, we find the user `vera`:

```text
vera:1000:...:1241186a4aac4f34f4bf7ace71b396a8:::
```

The obtained NT hash:

```text
1241186a4aac4f34f4bf7ace71b396a8
```

However, the NT hash alone is not enough to decrypt the local DPAPI masterkeys.

---

## 2. Recovering the Windows Password

For local DPAPI, we need the user's password.

Save the NT hash in a format suitable for John the Ripper:

```bash
echo 'vera:$NT$1241186a4aac4f34f4bf7ace71b396a8' > nt.john
```

Run a wordlist attack using `rockyou.txt`:

```bash
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john
```

John successfully recovers the password:

```text
minivera
```

This is Vera's Windows account password. It is not yet the password for the container.

---

## 3. Decrypting the DPAPI Masterkey

Return to the root of the KAPE image:

```bash
cd ../../../../..
```

Locate the user's DPAPI masterkey:

```text
C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a
```

Set the path to the masterkey and the user's SID:

```bash
MK="C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a"
SID="S-1-5-21-2529683458-431225740-1723070931-1000"
```

Now use `impacket-dpapi`:

```bash
impacket-dpapi masterkey -file "$MK" -sid "$SID" -password minivera
```

The masterkey is successfully decrypted:

```text
Decrypted key: 0x5e5715ec...9d40
```

Full key value:

```text
5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40
```

---

## 4. Extracting the Saved Password from Chrome

Now we move on to Chrome.

Saved credentials are stored in the SQLite database:

```text
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data
```

Copy it to a temporary directory:

```bash
cp "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" /tmp/LoginData
```

The Chrome encryption key is stored in:

```text
C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State
```

The value we are interested in is:

```text
os_crypt.encrypted_key
```

This key is additionally protected by Windows DPAPI. Therefore, we first use the masterkey we obtained earlier.

The following Python script can be used for decryption:

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

We get:

```text
VeraSecretVault => Wh4t1sV3raD0inG0nTh1sH0st
```

Thus, we finally have the password for the next stage — the VeraCrypt container.

---

## 5. Opening the VeraCrypt Container

The file:

```text
C/Users/vera/Documents/backup
```

is a VeraCrypt container.

It does not have to be opened through the GUI. `cryptsetup` can work with VeraCrypt-compatible containers:

```bash
sudo cryptsetup open --type tcrypt --veracrypt "C/Users/vera/Documents/backup" veracnt
```

Use the following as the passphrase:

```text
Wh4t1sV3raD0inG0nTh1sH0st
```

Create a mount point:

```bash
sudo mkdir -p /mnt/vera
```

Mount the container in read-only mode:

```bash
sudo mount -o ro /dev/mapper/veracnt /mnt/vera
```

After opening the container, we find the directory:

```text
secret_financial_documents/
```

It contains:

```text
transactions_q3.csv
important_invoice_byte_lotus.pdf
```

---

## 6. Finding the Flag in the PDF

A regular `pdftotext` will not help here because the PDF contains an image rather than ordinary text.

Therefore, extract the embedded images:

```bash
pdfimages -all "/mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf" /tmp/img
```

Then open the resulting image:

```text
/tmp/img-000.png
```

The required flag is visible in the image.

## Flag

```text
THM{1t_w4s_V3r4_A11_Al0ng?!}
```

---

## Full Set of Commands

For convenience, the entire process:

```bash
cd KAPE/C/Windows/System32/config

# Get Vera's NT hash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Recover the Windows password
echo 'vera:$NT$1241186a4aac4f34f4bf7ace71b396a8' > nt.john
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john

cd ../../../../..

MK="C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a"
SID="S-1-5-21-2529683458-431225740-1723070931-1000"

# Decrypt the DPAPI masterkey
impacket-dpapi masterkey -file "$MK" -sid "$SID" -password minivera

# Copy the Chrome database
cp "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data" /tmp/LoginData

# After obtaining the Chrome AES key, decrypt Login Data
# → VeraSecretVault => Wh4t1sV3raD0inG0nTh1sH0st

# Open the VeraCrypt container
sudo cryptsetup open --type tcrypt --veracrypt \
    "C/Users/vera/Documents/backup" veracnt

sudo mkdir -p /mnt/vera
sudo mount -o ro /dev/mapper/veracnt /mnt/vera

# Extract images from the PDF
pdfimages -all \
    "/mnt/vera/secret_financial_documents/important_invoice_byte_lotus.pdf" \
    /tmp/img

# The flag is located in /tmp/img-000.png
# THM{1t_w4s_V3r4_A11_Al0ng?!}

# Cleanup
sudo umount /mnt/vera
sudo cryptsetup close veracnt
```

## Key Takeaways

* DPAPI artifacts can be investigated offline if the masterkeys and the necessary data for decrypting them are available.
* In this case, the Chrome chain is `Local State → DPAPI → AES key → Login Data → AES-GCM`.
* A file without a header, with a size matching typical container sizes and high entropy, should be checked for an encrypted container.
* If `pdftotext` produces no output, that does not necessarily mean the PDF is empty: the content may be stored as embedded images.
* When working with flags, carefully check leetspeak: `V3r4`, `Al0ng`, and other substitutions may be significant.
