# A rat by any other name

**Category:** Crypto

**Flag:** `cdctf{Jaqurtis}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** MD5 digest `fe00ab6a1d242513c9f246344bf7da1d`. The prompt restricts the password to the rat's name: one initial uppercase letter, followed only by lowercase letters, total length at most eight. Common character-name and given-name dictionaries did not match the digest in an initial search, so search the stated shape directly. For example, Hashcat mode 0 is raw MD5 and attack mode 3 is a mask search:

## Solution

```text
hashcat -m 0 -a 3 --increment --increment-min=1 --increment-max=8 \
  fe00ab6a1d242513c9f246344bf7da1d '?u?l?l?l?l?l?l?l'
```

The matching eight-character name is `Jaqurtis`. Verify it separately with `hashlib.md5(b"Jaqurtis").hexdigest()`, which returns the supplied digest.

## Flag

```text
cdctf{Jaqurtis}
```
