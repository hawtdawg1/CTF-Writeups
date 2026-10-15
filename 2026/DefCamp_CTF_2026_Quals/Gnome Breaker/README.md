# Gnome Breaker

**Category:** Forensics  
**Difficulty:** Medium  
**Author:** Dodu Andrei  
**Artifact:** `gnome_rootfs.zip`  
**Flag:** `ECSC{C0ngrats_you_are_AGnomeFan}`

## Challenge Overview

The archive is a small fake Ubuntu root filesystem. The important directory is the user's GNOME keyring folder:

```text
home/masquerade/.local/share/keyrings/
├── default
├── default.keyring
├── login.keyring
└── user.keystore
```

I extracted the real encrypted payload from `login.keyring`, converted it to John the Ripper's `$keyring$` format, and cracked its weak master password with `rockyou.txt`:

```text
masquerade2k     (login.keyring)
```

Decrypting the keyring revealed a GNOME note named `Flag` containing:

```text
ECSC{C0ngrats_you_are_AGnomeFan}
```

---

## 1. Initial triage

I started by listing the archive instead of opening files one at a time:

```bash
unzip -l gnome_rootfs.zip
```

Most of the filesystem looked deliberately noisy: synthetic logs, Lorem Ipsum-style notes, tiny placeholder images, and random-looking binary files. One path immediately matched the wording of the challenge, though:

```text
home/masquerade/.local/share/keyrings/
```

GNOME Keyring is where a Linux desktop commonly stores secrets such as website logins, Wi-Fi credentials, tokens, and secure notes. That made the two `.keyring` files much more interesting than the fake logs.

The user's shell history also reinforced this direction:

```text
ls
cd .local/share/keyrings
xxd login.keyring | head
nmcli connection show
cat /etc/os-release
sudo apt update
htop
```

The original user had inspected `login.keyring` and then checked NetworkManager connections. That was a much stronger clue than the many occurrences of words such as `password` and `hint` scattered through the generated text files.

## 2. Identifying the keyring files

Both keyring files begin with the legacy GNOME Keyring signature:

```text
GnomeKeyring\n\r\0\n
```

For example, the beginning of `login.keyring` is:

```text
47 6e 6f 6d 65 4b 65 79 72 69 6e 67 0a 0d 00 0a
```

The two files parsed as follows:

| File | Label | Iterations | Salt | Items | Encrypted bytes |
|---|---:|---:|---|---:|---:|
| `login.keyring` | `Login` | 1603 | `4a24e2c4bd9f555c` | 1 | 160 |
| `default.keyring` | `Default` | 900 | `d272cc2faa9d5690` | 1 | 208 |

The unencrypted, hashed metadata gave one more useful detail. The item in `login.keyring` had this schema hash:

```text
f1b0a5949a0ad7a6162eb5fdce543a40
```

That is the MD5 hash of:

```text
org.gnome.keyring.Note
```

So `login.keyring` contained a secure note. That sounded very likely to be either the flag or a password leading to it.

## 3. A converter pitfall

John the Ripper includes `keyring2john.py`, but the bundled converter assumes a fixed offset after the keyring header. That did not work correctly for these files because they contain variable-length hashed item metadata before the encrypted data.

The incorrect conversion produced only 16 bytes such as:

```text
00000000000000010000000a7864673a
```

Those bytes are not ciphertext. They are unencrypted structure fields: the item count, item ID, and the beginning of the `xdg:schema` attribute.

Running John against that value could never recover the real password. This was an easy trap because John accepted the line as syntactically valid and simply reported zero guesses.

The correct approach was to parse the format rather than assume an offset. The relevant layout is:

```text
16 bytes  magic
4 bytes   major/minor/crypto/hash versions
variable  length-prefixed keyring label
16 bytes  modified and created timestamps
4 bytes   flags
4 bytes   lock timeout
4 bytes   hash iteration count
8 bytes   salt
16 bytes  reserved fields
variable  item count and hashed item metadata
4 bytes   encrypted-data length
variable  AES-CBC ciphertext
```

All integer fields are big-endian, and strings are stored as a 32-bit length followed immediately by that many bytes. They are not padded to a four-byte boundary.

After parsing the variable-sized metadata, the encrypted portion of `login.keyring` begins at offset 155 and is 160 bytes long.

## 4. Building the correct John hash

John's GNOME Keyring format is:

```text
$keyring$<salt>*<iterations>*<ciphertext_length>*0*<ciphertext_hex>
```

For `login.keyring`, the corrected line was:

```text
login.keyring:$keyring$4a24e2c4bd9f555c*1603*160*0*113e63748682be6fdf84e270e02843856ac6782c83c242fef58a474ef0d1046860c1913e36a0763a99188f6e2c976d9f6d0c43d5549b5c9d26247fec49ffa7e29baae9adf069113fa77e7edc32a5cb7c3dbf0889f2540dd79cc9ab93f37b27df17baf6ff4c97ec586e3e793132f0cc0e2e35c672ccbba4b7e1ba2fb6116670fab6396ceeba8941d5d706917cc442a55b8c753661fefb68e14f2a4f4e75a978e4
```

I saved that line to `keyring_hashes.txt` and ran a wordlist attack:

```bash
john --format=keyring --wordlist=rockyou.txt keyring_hashes.txt
```

John found the password:

```text
masquerade2k     (login.keyring)
```

This fits the challenge hint nicely: it is based on the username `masquerade`, with a weak suffix.

## 5. How the decryption works

The legacy GNOME Keyring format uses a custom repeated SHA-256 derivation followed by AES-128-CBC.

Given the recovered password and the eight-byte salt:

1. Calculate `SHA256(password || salt)`.
2. Hash the resulting 32-byte digest repeatedly until the configured iteration count is reached.
3. Use the first 16 digest bytes as the AES-128 key.
4. Use the final 16 digest bytes as the CBC IV.
5. Decrypt the encrypted extent with AES-128-CBC.

In Python, the essential operation is:

```python
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

password = "masquerade2k"
salt = bytes.fromhex("4a24e2c4bd9f555c")
iterations = 1603

digest = hashlib.sha256(password.encode() + salt).digest()
for _ in range(1, iterations):
    digest = hashlib.sha256(digest).digest()

key = digest[:16]
iv = digest[16:32]

decryptor = Cipher(algorithms.AES(key), modes.CBC(iv)).decryptor()
plaintext = decryptor.update(ciphertext) + decryptor.finalize()
```

There is also a reliable correctness check built into the format. The first 16 decrypted bytes must equal the MD5 digest of all remaining decrypted bytes:

```python
assert plaintext[:16] == hashlib.md5(plaintext[16:]).digest()
```

That assertion passed, so the recovered password and plaintext were not false positives.

## 6. Reading the decrypted entry

Extracting printable strings from the verified plaintext produced:

```text
Flag
ECSC{C0ngrats_you_are_AGnomeFan}
xdg:schema
org.gnome.keyring.Note
```

The internal structure confirmed that:

- the item's display name is `Flag`;
- its secret is the ECSC flag;
- its schema is `org.gnome.keyring.Note`.

The separate `default.keyring` contained Chrome-related metadata, but it was not necessary to solve the challenge. The flag was already stored directly in the login keyring's secure note.

## 7. Dead ends and what actually mattered

The archive contains a lot of material designed to slow down triage:

- logs full of random words such as `password` and `hint`;
- `archive_9d3b.bin`, `tmp_45a9.dat`, and `user.keystore`, which look high-entropy;
- one-pixel PNG files;
- ordinary system configuration files and placeholder documents.

I initially checked those files for plaintext strings, UTF-16 text, obvious XOR relationships, and password hints. None of them led to the flag. The useful evidence was much simpler:

1. The challenge description mentioned a file that stores Wi-Fi and login passwords.
2. The GNOME keyring directory existed in the supplied home folder.
3. `.bash_history` showed that the user had inspected `login.keyring`.
4. Its hashed schema identified the stored object as a GNOME secure note.
5. The weak password was present in `rockyou.txt`.

The main technical obstacle was not the password attack itself; it was extracting the correct ciphertext from a variable-length keyring structure.

## Flag

```text
ECSC{C0ngrats_you_are_AGnomeFan}
```
