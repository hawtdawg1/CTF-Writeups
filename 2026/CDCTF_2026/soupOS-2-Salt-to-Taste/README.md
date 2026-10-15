# soupOS 2 - Salt to Taste

**Category:** Pwn

**Flag:** `cdctf{salt_to_taste_any_preimage_works}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

Five soupOS stages share one challenge instance, a 32-bit OS source handout, and a kernel symbol map.

## Solution

The readable kitchen account record exposed headchef's AlphaSOUP-32 hash `f63a9eb7`. `alphasoup.c` (`soupos2/soupos/src/alphasoup.c`) defines the 32-bit, noncryptographic hash, and a preimage search verified a matching input. The instance accepted a matching secret for the headchef-only special and displayed `cdctf{salt_to_taste_any_preimage_works}`.

## Flag

```text
cdctf{salt_to_taste_any_preimage_works}
```
