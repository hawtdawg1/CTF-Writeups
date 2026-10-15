# Doomscroll Hell

**Category:** Forensics

**Flag:** `cdctf{doomscrolling_is_so_much_fun!}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** `doomscroll.zip` with 15 MP4 reels. Unzip it and inspect the MP4 box layout and file tails. Each reel has a trailing `uuid` box: a four-byte big-endian box length, the four-byte type `uuid`, a 16-byte UUID, and a payload. In these files the box length is 92 or 93 bytes, and the data after the 24-byte header is a PDF fragment. Extract each payload after its 24-byte box header.

## Solution

The fragments containing the PDF header and `stream\n` go first, in this order: `CWBRyIGYAOd`, `CFH95xUXh5X`, `C_pn-Ww2wKg`, `C4cE45r8ICF`, `CyzAgQA4oVO`, `Cks57_V-Hsy`. The fragments containing `\nendstream` and the PDF trailer go last: `C1-sQVqsTIu`, `CdWtouUG4Bu`, `C9nruRG2aNs`. The six fragments between them are compressed stream bytes, so they cannot be ordered visually. Try their 720 permutations and retain the one for which `zlib.inflate` succeeds. The valid middle order is `CU4g7CDhW5q`, `CDx8NCxzTUA`, `CtJum68xxwr`, `CPZGeCCcZfB`, `Ct9t7nQO6KL`, `CYzUFRaPz5o`. Append `.mp4` to each identifier to find the reel.

Concatenating the 15 payloads in this order reconstructs a PDF. Check the fragment order by testing for a valid zlib stream, then inflate the PDF page content. In that stream, `Retention key:` is immediately followed by `cdctf{doomscrolling_is_so_much_fun!}`. The zlib check has exactly one valid middle permutation, and the recovered PDF and text provide independent readable validation.

## Flag

```text
cdctf{doomscrolling_is_so_much_fun!}
```
