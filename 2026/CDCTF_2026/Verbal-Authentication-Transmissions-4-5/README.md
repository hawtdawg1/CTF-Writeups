# Verbal Authentication Transmissions 4/5

**Category:** Crypto

**Flag:** `cdctf{idontcare1}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

An audio recording reads eight numbered six-word S/Key one-time password codes.

## Solution

1. Listen to `OTP_CODES_VAT.mp3`. It reads eight numbered codes, each six short words long. The format is the RFC 2289/S/Key dictionary representation of a 64-bit key: six 11-bit word indexes carry 64 data bits plus a two-bit checksum. An S/Key word-list decoder checks the words and their two-bit checksums. The recovered code sequence, from 1 through 8, is:

   | # | Six words |
   | -: | - |
   | 1 | `BAIT GAUR OS TACT TIP DUD` |
   | 2 | `GLEN YAWL PEW RUNG BUOY BODY` |
   | 3 | `GRUB TRY BABY MUFF GRIM ACME` |
   | 4 | `CUB CUTS GAIN SLUM MUST ELM` |
   | 5 | `RUST HERB BAIL SAVE IFFY DO` |
   | 6 | `REAR NOLL MART TREE WORM WAVE` |
   | 7 | `MIST LOG HOST SORT HOLT ACRE` |
   | 8 | `INCH WERT IKE LOS DRAB SIS` |

2. Decode code 8 to the eight bytes `9e9f487592d76678`. The words' two-bit checksum distinguishes close-sounding alternatives. Comparing codes 7 and 6 reveals the chain rule: take the **lowercase hexadecimal text** of the prior eight-byte key, MD5-hash that ASCII text, then XOR the first and last eight bytes of the 16-byte digest. `vat4_chain_hex.py` implements this exact step. Applied to code 8, it produces code 7 (`bca4ae6c71599c90`); repeated application produces all earlier codes. The ASCII-hex input is important: MD5 of the raw eight bytes does not reproduce the chain.

3. Crack only the initial password against code 8's bytes. Compute MD5 of a candidate password's UTF-8 bytes, XOR its digest halves, and compare with `9e9f487592d76678`. A candidate search found one hit in its first million candidates: `idontcare1`. Its folded MD5 is code 8, and following the chain reproduces all seven earlier word encodings exactly. Wrap the password as `cdctf{idontcare1}`.

## Flag

```text
cdctf{idontcare1}
```
