# soupOS 3

**Category:** Pwn

**Flag:** `cdctf{bad_recipe_the_loader_reads_wide}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

Five soupOS stages share one challenge instance, a 32-bit OS source handout, and a kernel symbol map.

## Solution

The challenge build of the ELF loader omits a file-offset bound check that appears in the ordinary source path. The symbol map and two test ELF files document the loader behavior: a 212-byte compact file and a separate 378-byte leak-scanning file. The challenge instance output was `cdctf{bad_recipe_the_loader_reads_wide}`.

## Flag

```text
cdctf{bad_recipe_the_loader_reads_wide}
```
