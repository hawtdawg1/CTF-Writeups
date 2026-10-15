# What is the flag

**Category:** Crypto

**Flag:** `cdctf{it is sure where it isn't, within reason, and it knows where it was}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

The prompt supplies hexadecimal ciphertext and asks for the plaintext in `cdctf{...}` form.

## Solution

**Starting material:** the hexadecimal ciphertext in the prompt, beginning `9c9b9c8b9984...`, and a request to wrap the plaintext in `cdctf{...}`. Decode pairs of hex digits to bytes. XOR every byte with `0xFF`, equivalent to taking the one's complement of each byte. The beginning `9c 9b 9c 8b 99 84` becomes the ASCII string `cdctf{`; decoding the whole sequence yields the supplied flag itself. A direct check is `bytes(b ^ 0xff for b in bytes.fromhex(hex_text)).decode()` where `hex_text` is the entire prompt string.

## Flag

```text
cdctf{it is sure where it isn't, within reason, and it knows where it was}
```
