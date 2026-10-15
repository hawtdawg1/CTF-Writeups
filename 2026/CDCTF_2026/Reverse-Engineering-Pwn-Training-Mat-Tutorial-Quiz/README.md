# Reverse Engineering & Pwn Training Mat - Tutorial Quiz

**Category:** Pwn / Reverse Engineering

**Flag:** `cdctf{R2vv1n_!t_UP!}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

The challenge statement explicitly requires **three separate flags**. Begin with the two attachments, `skippy.py` and `static`, then complete the tutorial quiz linked from the challenge.

## Solution

Work through the training page using those file findings. The quiz used the Skippy output variable name `test_result_del_for_prod` and the Static facts that the required length is `24` and the comparison operation is `XOR`. Completing the quiz displayed its own third flag, `cdctf{R2vv1n_!t_UP!}`. This flag came from the page, while the first two came from the supplied attachments.

## Flag

```text
cdctf{R2vv1n_!t_UP!}
```
