# Makmal Buta, UM

**Category:** Crypto
**Flag:** `UMCS{NMBRXD3}`

## Challenge Overview

> *"The following message was recovered:"* [scrambled ciphertext shown in challenge description]

Two files were given: `loganathan_translated_to_py3.py` (a Python script) and `output.txt` (a log containing 637 sequential hexadecimal IDs).

## Initial Enumeration / Recon

Opening `output.txt` shows 637 lines, each containing a hex value of the form `0xXXXXXXXX`. Most entries have the status label "Archive Entry Encrypted", while a handful contain cryptic hints like "Caesar Shift", "Fibo nasi", "prime factors", and "LSB of X matches ADC output" — possible misdirection.

Reading `loganathan_translated_to_py3.py` reveals the actual logic:

- It extracts the IDs using a regex pattern.
- It mentions "predicting" and uses the XOR operator (`^`).
- It loops through the IDs and processes them.

The combination of 32-bit hex IDs and the word "predicting" immediately points to **Mersenne Twister (MT19937)** — Python's default PRNG.

## Vulnerability Analysis

MT19937 has a known property: if you observe **624 consecutive 32-bit outputs**, you can fully reconstruct the internal state and predict all subsequent outputs.

With 637 IDs available, the first 624 recover the state, and the remaining 13 are XOR-encrypted flag characters:

```
ID[i] = random.getrandbits(32) XOR flag_char[i]
```

## Exploitation Steps

```python
import re
import random

def unshiftRight(x, shift):
    res = x
    for _ in range(32):
        res = x ^ (res >> shift)
    return res

def unshiftLeft(x, shift, mask):
    res = x
    for _ in range(32):
        res = x ^ ((res << shift) & mask)
    return res

def untemper(v):
    v = unshiftRight(v, 18)
    v = unshiftLeft(v, 15, 0x9d2c5680)
    v = unshiftLeft(v, 7,  0x9908b0df)
    v = unshiftRight(v, 11)
    return v

with open("output.txt", "r") as f:
    content = f.read()

ids_hex = re.findall(r"ID: (0x[0-9a-fA-F]{8})", content)
ids     = [int(x, 16) for x in ids_hex]

state = [untemper(x) for x in ids[:624]]
state.append(624)
random.setstate((3, tuple(state), None))

encrypted = ids[624:]
room_number = b""
for enc in encrypted:
    pred = random.getrandbits(32)
    dec  = enc ^ pred
    room_number += dec.to_bytes(4, "big")

flag = room_number.replace(b"\x00", b"").decode("utf-8")
print(f"Flag: {flag}")
```

```
Flag: UMCS{NMBRXD3}
```

## Flag

```
UMCS{NMBRXD3}
```

Python's `random` module should never be used for anything security-sensitive. MT19937 is cryptographically weak — 624 outputs is all an attacker needs to own the state entirely.
