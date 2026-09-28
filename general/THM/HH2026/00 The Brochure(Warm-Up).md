**Tags:** OSINT, Web.

**Difficulty:** Easy.

### 1. Downloading the brochure image

The archive contained a file named **thebrochure.png**.

### 2. Finding the hidden Instagram account

**Command:**
We look at the brochure image — it contains the text “Find us on Instagram”.

**Result:**
The account is — [https://www.instagram.com/thebytelotusresort](https://www.instagram.com/thebytelotusresort)

### 3. Finding the hidden connection

This account has only one following — [https://www.instagram.com/veratheconcierge](https://www.instagram.com/veratheconcierge)

### 4. Extracting the flag using igviewer.net

We open the profile through the service [https://igviewer.net/profile/veratheconcierge](https://igviewer.net/profile/veratheconcierge)

**Result:**
There are only 3 posts.
The descriptions of all three posts contain parts of **Base64** (encoded in separate parts).

### 5. Combining and decoding B64

```bash
echo "VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==" | base64 -d
```

**Command output:**

```text
THM{V3r@s_aCC0unt_h4s_b33n_f0und!}
```

**Flag found!**
