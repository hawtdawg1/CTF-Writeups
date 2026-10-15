# soupOS 4

**Category:** Pwn

**Flag:** `cdctf{too_many_cooks_ring0_from_a_script}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

Five soupOS stages share one challenge instance, a 32-bit OS source handout, and a kernel symbol map.

## Solution

SoupyC's array storage sits next to an otherwise unused post-run hook, and its index assignment does not reject a negative index in the challenge build. The symbol map names `serve_the_special`; the challenge instance reached it through the array bug and printed `cdctf{too_many_cooks_ring0_from_a_script}`.

## Flag

```text
cdctf{too_many_cooks_ring0_from_a_script}
```
