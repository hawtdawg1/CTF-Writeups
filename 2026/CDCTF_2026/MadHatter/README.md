# MadHatter

**Category:** Crypto

**Flag:** `cdctf{We're_@LL_Mad_h3rE!}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** `hatter.7z`, containing `key.txt` and the 32-byte `super_secret.txt`. The 64-character contents of `key.txt` look hexadecimal, but the successful derivation uses those characters as passphrase **text**, after removing the line ending; it does not hex-decode them.

## Solution

Compute `key = SHA256(passphrase)` and `iv = SHA256(key || passphrase)[:16]`. Decrypt `super_secret.txt` with AES-256-CBC and remove PKCS#7 padding. The result is `ini~lq]o-xoUJFFUGknUb9xO+w\n`, which is not yet the flag. XOR each byte before the newline with `0x0A`; the prefix becomes `cdctf{`, and the full result is `cdctf{We're_@LL_Mad_h3rE!}`.

```python
from hashlib import sha256
from pathlib import Path
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives.padding import PKCS7

folder = Path("hatter_research/hatter")
phrase = (folder / "key.txt").read_bytes().strip()
ciphertext = (folder / "super_secret.txt").read_bytes()
key = sha256(phrase).digest()
iv = sha256(key + phrase).digest()[:16]
dec = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
padded = dec.update(ciphertext) + dec.finalize()
unpad = PKCS7(128).unpadder()
inner = unpad.update(padded) + unpad.finalize()
flag = bytes(b ^ 10 for b in inner.rstrip(b"\n"))
print(flag.decode())
```

Reversing these operations reproduces `super_secret.txt` byte for byte, validating the layer order and exact flag.

## Flag

```text
cdctf{We're_@LL_Mad_h3rE!}
```
