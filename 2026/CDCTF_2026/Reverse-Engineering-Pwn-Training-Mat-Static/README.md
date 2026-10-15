# Reverse Engineering & Pwn Training Mat - Static

**Category:** Reverse Engineering

**Flag:** `cdctf{AllXOR_Nev3rLupus}`

**Verification:** Derived from the supplied Static binary; checker acceptance was not documented.

## Challenge Overview

The challenge statement explicitly requires **three separate flags**. Begin with the two attachments, `skippy.py` and `static`, then complete the tutorial quiz linked from the challenge.

## Solution

The 64-bit ELF `static` has symbols for two 24-byte arrays, `cheque` and `key`. Its `main` first requires a 24-character argument, then compares `argument[i] ^ key[i]` with `cheque[i]` for each index from 0 to 23. Reverse that equality byte by byte: `argument[i] = cheque[i] ^ key[i]`. The arrays are `cheque = 676c6c647151497c727861064f6e59366fda6c3508f5cb82` and `key = 04080f10172a08101e202e5410203c405ca820407880b8ff`. Their XOR decodes to `cdctf{AllXOR_Nev3rLupus}`; XORing that result back with `key` reproduces every `cheque` byte.

## Flag

```text
cdctf{AllXOR_Nev3rLupus}
```
