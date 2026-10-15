# Blake's Bride

**Category:** Forensics / Crypto

**Flag:** `cdctf{epithalamium738-trousseau201-honeymoon069}`

**Blood award:** 2nd blood

**Verification:** Recovered or checked against supplied material.

## Challenge Overview

**Starting material:** the supplied image (`blake.png`) and the clue that three wedding-related passwords each consist of one word followed by three digits. Inspect the image metadata as well as its pixels. Its `XML:com.adobe.xmp` field contains three 128-hex-character strings, in password order. Their length is consistent with 512-bit BLAKE2b; the wording “to be” and “Blake” signals that algorithm.

## Solution

Build a wedding vocabulary that includes specialized terms such as *epithalamium* (wedding poem or song), *trousseau* (bridal outfit and possessions), and *honeymoon*. For each word, test suffixes `000` through `999` with `hashlib.blake2b(candidate.encode()).hexdigest()`. The matches, in the metadata's order, are `epithalamium738`, `trousseau201`, and `honeymoon069`. A compact independent check is:

```python
from PIL import Image
from hashlib import blake2b
import re

xmp = Image.open("blake.png").info["XML:com.adobe.xmp"]
hashes = re.findall(r"(?<![0-9a-f])[0-9a-f]{128}(?![0-9a-f])", xmp)
words = ["epithalamium738", "trousseau201", "honeymoon069"]
assert len(hashes) == 3
assert all(blake2b(w.encode()).hexdigest() == h for w, h in zip(words, hashes))
print("cdctf{" + "-".join(words) + "}")
```

All three complete hashes match, which checks spelling, padding of the numeric suffix, and order.

## Flag

```text
cdctf{epithalamium738-trousseau201-honeymoon069}
```
