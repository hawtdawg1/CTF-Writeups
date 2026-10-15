# Low On Fun

**Category:** Reverse Engineering

**Flag:** `cdctf{Y_m4ny_functi0s_wh3n_f3w_d0_trick}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

The supplied `low_on_fun.py` appears to contain an empty validator, but its real logic is hidden in marshalled code objects.

## Solution

Start with the supplied `low_on_fun.py`. The visible `changing()` function is empty, but `main()` splits the large `functions` byte string at `DELIM`, calls `marshal.loads` on each piece, and assigns each resulting code object to `changing.__code__`. Thus the same function first reads input, then checks its format, then validates it. Reading the visible function body alone misses the flag check.

Extract the `functions` assignment as a Python literal with `ast.literal_eval`, split it into three blobs, and inspect the third marshalled code object's constants and original serialized bytecode. The validator checks a 40-byte input and implements RC4. Its first long hex constant is the 128-byte RC4 key. The 40-byte comparison target is assembled from the subsequent two-character hex constants in the order used by the bytecode; in this analysis, those indices came from `raw = part[26:26 + len(code.co_code)]` and `code.co_consts[raw[i+1]]` for `i = 68, 72, ..., 224`. RC4 is symmetric, so decrypt that target with the embedded key. This yields `cdctf{Y_m4ny_functi0s_wh3n_f3w_d0_trick}`. As a check, the recovered string is 40 bytes, starts with `cdctf`, and RC4-encrypting it with the same key exactly reproduces the 40-byte embedded target.

## Flag

```text
cdctf{Y_m4ny_functi0s_wh3n_f3w_d0_trick}
```
