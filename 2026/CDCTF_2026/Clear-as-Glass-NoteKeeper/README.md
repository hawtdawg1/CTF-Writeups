# Clear as Glass (NoteKeeper)

**Category:** Forensics / Reverse Engineering

**Flag:** `cdctf{G1455w0rM_w4s_pr3tTy_c0oL}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** the NoteKeeper source repository and its history, specifically `notekeeper-history/app.py`. A suspicious commit (`bd75399`) added `notes_list = "..."`, a loop that subtracts `0xE0000` from the code point of each character in that string, and `exec(tags_list)`. Terminal rendering makes the string appear empty, but a code-point inspection shows that it contains 395 Unicode tag characters; the first is `U+E000A`.

## Solution

Read the string from the source as text, decode the tag characters, and **inspect the result without importing or executing the app**:

```python
from pathlib import Path
source = Path("notekeeper-history/app.py").read_text(encoding="utf-8")
hidden = source.split('notes_list = "', 1)[1].split('"', 1)[0]
payload = "".join(chr(ord(ch) - 0xE0000) for ch in hidden)
print(payload)
```

The decoded text declares two hex strings and computes `fl = bytes(a ^ b for a, b in zip(k, f))`. It also constructs a download URL and invokes `curl`; that side effect is not needed to recover the flag. Copy only the two hex constants into a separate static calculation:

```python
f = bytes.fromhex("04266506711a20733247221657304b2d6055141d76002415333b5911270e2b3f")
k = bytes.fromhex("6742067217616742067217616742067217616742067217616742067217616742")
print(bytes(a ^ b for a, b in zip(k, f)).decode())
```

This prints `cdctf{G1455w0rM_w4s_pr3tTy_c0oL}`. The commit history and the static decode together explain how the hidden code was inserted and what it contains.

## Flag

```text
cdctf{G1455w0rM_w4s_pr3tTy_c0oL}
```
