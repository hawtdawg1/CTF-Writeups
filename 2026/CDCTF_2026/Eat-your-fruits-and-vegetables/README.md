# Eat your fruits and vegetables!

**Category:** Crypto

**Flag:** `cdctf{Carrot}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

**Starting material:** `data.db.enc`, 960,000 bytes. The prompt says there are 30,000 records of 32 bytes each. Within a record, four eight-byte fields are ordered as name, car, produce, and OS. Under AES-ECB, each 16-byte ciphertext block is deterministic: the first block represents `name || car`, and the second represents `produce || OS`. No AES key is required for this analysis.

## Solution

Split the file into 32-byte records and count which second blocks follow each first block. There are 36 distinct first blocks and 18 distinct second blocks, consistent with 12 names times three car brands and six produce choices times three OS choices. Exactly three first blocks each connect to only the same three second blocks; these are the three car variants for Bob, whose produce choice is constant according to the clue. The three associated second blocks appear 1,768, 1,780, and 1,852 times globally: 5,400 records, or 18% of 30,000. The prompt assigns the 18% frequency to Carrot.

```python
from pathlib import Path
from collections import Counter, defaultdict

data = Path("data.db.enc").read_bytes()
assert len(data) == 30_000 * 32
rows = [(data[i:i+16], data[i+16:i+32]) for i in range(0, len(data), 32)]
next_blocks = defaultdict(set)
for first, second in rows:
    next_blocks[first].add(second)
bob_car_blocks = [first for first, following in next_blocks.items() if len(following) == 3]
assert len(bob_car_blocks) == 3
bob_produce_os = next_blocks[bob_car_blocks[0]]
assert all(next_blocks[first] == bob_produce_os for first in bob_car_blocks)
counts = Counter(second for _, second in rows)
assert sorted(counts[block] for block in bob_produce_os) == [1768, 1780, 1852]
assert sum(counts[block] for block in bob_produce_os) == 5400
```

This checks the grouping and frequency independently of the unknown ciphertext labels.

## Flag

```text
cdctf{Carrot}
```
