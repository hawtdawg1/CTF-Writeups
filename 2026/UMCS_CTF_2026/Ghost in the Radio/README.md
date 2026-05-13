# Ghost in the Radio [Got first blood for this :) ]

**Category:** Crypto

**Flag:** `UMCS{XOR_M4G1C_BYT3S}`

## Challenge Overview

![Challenge Description](GhostInTheRadio.png)

> *"During the UM Cybersecurity Summit, a strange signal was detected on the campus network. It contains a 'corrupted.bin' file. Initial analysis shows it's encrypted with a rolling XOR cipher. If you can't fix the 'drift' in the key, the image will remain a ghost."*
>
> **Hint:** *"XOR the first 4 bytes of your file with the standard PNG magic bytes. Do you notice the 'drift' in the resulting keys?"*

One file: `corrupted.bin`.

## Initial Enumeration / Recon

```bash
file corrupted.bin
# corrupted.bin: data
```

No recognisable magic bytes. Following the hint, XOR the first 4 bytes with the PNG magic (`\x89\x50\x4E\x47`):

```
corrupted[0] ^ 0x89 = 0x42  ('B')
corrupted[1] ^ 0x50 = 0x43  ('C')
corrupted[2] ^ 0x4E = 0x44  ('D')
corrupted[3] ^ 0x47 = 0x45  ('E')
```

The keys are `0x42, 0x43, 0x44, 0x45` — consecutive ASCII values. The "drift" the hint mentions is a rolling increment: the XOR key at byte position `i` is `(0x42 + i) % 256`.

## Exploitation Steps

```python
with open("corrupted.bin", "rb") as f:
    ciphertext = f.read()

plaintext = bytes(
    ciphertext[i] ^ ((0x42 + i) % 256)
    for i in range(len(ciphertext))
)

with open("recovered.png", "wb") as f:
    f.write(plaintext)
```

After decryption, `recovered.png` opens as a valid image. The flag is embedded visually inside the image.

## Flag

```
UMCS{XOR_M4G1C_BYT3S}
```

Known-plaintext attacks on XOR ciphers are trivially easy when you know the file type — PNG, PDF, ELF, and ZIP all have fixed magic bytes at the start. The rolling/incrementing key here is only marginally harder than a static key because the pattern is immediately obvious once you decode the first four bytes.
