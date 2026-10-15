# Script of Theseus

**Category:** Forensics

**Flag:** `cdctf{0D}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

**Starting material:** `original_epic_python_script.py` and `replaced_epic_python_script.py`. The challenge asks for one differing byte as uppercase hex, and the “Windows user” clue suggests line endings. Compare raw bytes, rather than relying on an editor that normalizes newline display.

## Solution

```python
from pathlib import Path
original = Path("original_epic_python_script.py").read_bytes()
replacement = Path("replaced_epic_python_script.py").read_bytes()
assert original.replace(b"\n", b"\r\n") == replacement
print(len(original), len(replacement))  # 156, 163
```

The seven LF (`0A`) newlines in the original have become CRLF (`0D 0A`) in the replacement, accounting exactly for the seven-byte increase. The added byte is hexadecimal `0D`.

## Flag

```text
cdctf{0D}
```
