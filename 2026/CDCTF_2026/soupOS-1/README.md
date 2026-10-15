# soupOS 1

**Category:** Pwn

**Flag:** `cdctf{mise_en_place_two_paths_one_check}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

Five soupOS stages share one challenge instance, a 32-bit OS source handout, and a kernel symbol map.

## Solution

`shell.c` (`soupos1/soupos/src/shell.c`) applies a file permission check when reading `FLAG1.TXT`, while `soupyc.c` (`soupos1/soupos/src/soupyc.c`) reaches the VFS read path without the same check. `challenge.c` (`soupos1/soupos/src/challenge.c`) identifies the file as headchef-owned mode 600. The challenge instance displayed `cdctf{mise_en_place_two_paths_one_check}` when that mismatch was exercised.

## Flag

```text
cdctf{mise_en_place_two_paths_one_check}
```
