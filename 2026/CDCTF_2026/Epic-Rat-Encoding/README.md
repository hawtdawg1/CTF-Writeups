# Epic Rat Encoding

**Category:** Crypto

**Flag:** `cdctf{Tom_Bevill_Building_at_noon_next_week_Thursday}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** eight decimal integers in the prompt, beginning `5576975263002879264` and ending `8247045712450168172`. Convert each integer to an unsigned eight-byte **big-endian** value, then concatenate the results in the given order:

## Solution

```python
numbers = [5576975263002879264, 7022273403317198932,
           8029109180213651820, 7791300427398276201,
           7955362869705469551, 8029390844169057312,
           8603394173939509365, 8247045712450168172]
message = b"".join(n.to_bytes(8, "big") for n in numbers)
print(repr(message))
```

The bytes read `Meet me at the Tom Bevill Building at noon next week Thursday\x00%l`. The terminal `\x00%l` is not part of the sentence; it is trailing data in the final eight-byte block. Extract the place and time, then follow the prompt's underscore-separated flag template and capitalization.

## Flag

```text
cdctf{Tom_Bevill_Building_at_noon_next_week_Thursday}
```
