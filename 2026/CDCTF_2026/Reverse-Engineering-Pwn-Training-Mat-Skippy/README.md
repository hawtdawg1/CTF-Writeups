# Reverse Engineering & Pwn Training Mat - Skippy

**Category:** Reverse Engineering

**Flag:** `cdctf{SkippingOnMyWay_}`

**Verification:** Derived from the supplied Skippy script; checker acceptance was not documented.

## Challenge Overview

The challenge statement explicitly requires **three separate flags**. Begin with the two attachments, `skippy.py` and `static`, then complete the tutorial quiz linked from the challenge.

## Solution

In `skippy.py`, `encrypt()` shuffles a 61-character alphabet but then overwrites positions `(A × i) mod 61` with successive secret characters. The known output is the 61-character `test_result_del_for_prod` string. The shuffle is irrelevant at those overwritten positions. Try the possible step values `A` and look for the required `cdctf{` prefix and closing brace; `A = 15` extracts `''.join(test_result_del_for_prod[(15*i) % 61] for i in range(23))`, giving `cdctf{SkippingOnMyWay_}`. Every extracted character matches its corresponding position in the provided encoded string.

## Flag

```text
cdctf{SkippingOnMyWay_}
```
