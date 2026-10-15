# Verbal Authentication Transmissions 1/5: Going HAM

**Category:** Crypto

**Flag:** `cdctf{N4T0_comms}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

A radio recording repeats a sequence of spoken hexadecimal byte values.

## Solution

1. Listen to `captured_radio.mp3`. The 97-second transmission repeats a sequence of spoken hexadecimal byte values. Record the bytes in order, using the repeated reading to settle ambiguous letters or digits.

2. The resulting hex stream is `63 64 63 74 66 7B 4E 34 54 30 5F 63 6F 6D 6D 73 7D`. Decode every pair as an ASCII byte. For example, `63 64 63 74 66 7B` is `cdctf{`, which confirms both byte alignment and the flag format. `bytes.fromhex('63 64 63 74 66 7B 4E 34 54 30 5F 63 6F 6D 6D 73 7D').decode('ascii')` yields `cdctf{N4T0_comms}`.

3. Compare the recovered bytes with the second spoken pass; the recording supplies redundancy.

## Flag

```text
cdctf{N4T0_comms}
```
