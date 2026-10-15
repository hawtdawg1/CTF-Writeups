# ¿Dónde está la biblioteca?

**Category:** Pwn

**Flag:** `cdctf{B3_c4r3fu1_w17h_SUID5!_657419873647598_b4c71ad6}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

**Starting material:** the prompt says a privileged job runs `root_calls_me.py` once a minute, redirects its output to `output.txt`, and gives the participant control over a `hashlib/` directory. Inspecting the script shows an import of `hashlib` before it reads the protected flag. Python resolves an import using its module search path, so the controlled directory can supply code under that name when the job starts. The redirected output is the observable evidence for what the privileged process did.

## Solution

In the challenge instance, the job's output yielded a flag. A first copy of the text used lowercase `l` in `c4r3ful` and was rejected. Inspecting `output.txt` byte for byte corrected that character to digit `1`, producing `c4r3fu1`. The corrected string matches the output and was accepted as the challenge flag.

## Flag

```text
cdctf{B3_c4r3fu1_w17h_SUID5!_657419873647598_b4c71ad6}
```
