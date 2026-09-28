**Tags:** OSINT, Hashing, Social Media.

**Difficulty:** Easy.

## 1. Obtaining the Room File

First, we obtain the archive:

```text
overheard-at-breakfast-1784259780309.zip
```

Inside the archive, there is an image:

```text
conversation.png
```

The image contains a conversation between **Ponzi** and **Lambo**.

---

## 2. Analyzing the Conversation

During the conversation, Ponzi tries to find Lambo's social media contact.

Lambo says that he barely uses social media nowadays, but previously used a free tool that allowed users to upload a profile and link other social accounts to it.

He also says:

> "Started with a G, if I remember correctly."

In addition, Lambo leaves his email address:

```text
lambobytelotushotel@gmail.com
```

The hint about the letter `G` and the ability to link a profile with other accounts suggests that the service is **Gravatar**.

---

## 3. Searching Gravatar by Email

Gravatar allows users to access an avatar using the MD5 hash of their email address.

The format used in the log is:

```text
https://www.gravatar.com/avatar/HASH_OF_EMAIL
```

Therefore, we first need to calculate the MD5 hash of:

```text
lambobytelotushotel@gmail.com
```

We run:

```bash
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

We get the hash:

```text
d4a5fc5d3128890778667e24617d7cc0
```

The Gravatar URL will then be:

```text
https://www.gravatar.com/avatar/d4a5fc5d3128890778667e24617d7cc0
```

---

## 4. Finding the Profile

Using the email address, we find the Gravatar profile with holehe:

```text
https://gravatar.com/cheerfullysongf28e3c3716
```

The profile belongs to:

```text
Lambo
```

The profile also contains:

```text
Lam-boh · Byte Lotus Hotel
```

Thus, the hint from the conversation indeed led us to Lambo's profile.

---

## 5. Obtaining the Flag

The discovered profile contains the following message:

```text
Funny thing about email hashes, they follow you places you didn't expect. Glad you found the right corner of the internet! Here is your prize: VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
```

The resulting string has the characteristic format of Base64.

We decode:

```text
VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
```

After decoding, we get:

```text
THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
```

---

## Conclusion

The solution chain:

```text
conversation.png
       │
       ▼
Conversation between Ponzi and Lambo
       │
       ▼
Email:
lambobytelotushotel@gmail.com
       │
       ▼
Hint "started with G"
       │
       ▼
Gravatar
       │
       ▼
MD5 email
d4a5fc5d3128890778667e24617d7cc0
       │
       ▼
Find Lambo's profile
       │
       ▼
Base64 string
VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
       │
       ▼
Base64 decode
       │
       ▼
THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
```
