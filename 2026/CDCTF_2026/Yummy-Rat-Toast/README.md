# Yummy Rat Toast

**Category:** Crypto

**Password:** `Alfredo Linguini01`

**Blood award:** 1st blood

**Verification:** Password matches the supplied MD5 digest.

## Challenge Overview

**Starting material:** MD5 digest `3f1ebefc63dc39f3c9b934a30accb221`. Remy, “Anyone can cook,” and his friend identify *Ratatouille*. The name most relevant to Remy's cooking partnership is Alfredo Linguini. Test the full name, including the space, and ordinary password suffixes. The matching candidate has `01` appended to the full name:

## Solution

```python
from hashlib import md5
password = "Alfredo Linguini01"
assert md5(password.encode()).hexdigest() == "3f1ebefc63dc39f3c9b934a30accb221"
print(password)
```

The capitalization, space, and two-digit suffix matter. The captured prompt asked for the password, not a `cdctf{...}` wrapper, so the answer is shown as a password.

## Password

```text
Alfredo Linguini01
```
