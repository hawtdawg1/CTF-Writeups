# Crazy?

**Category:** Forensics

**Flag:** `cdctf{1_w@5_cr@zy_0nc3}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

A large text file repeats a meme phrase thousands of times, with one altered occurrence.

## Solution

1. Open `text.txt`, a 496,800-byte file dominated by the repeated `Crazy? I was crazy once. They put me in a room...` meme loop. Search for characters outside the ordinary phrase or for the leetspeak marker `@`.

2. At character offset `297116`, one iteration changes the usual words to `1 w@5 cr@zy 0nc3`. The analysis counted this exact mutation once, versus 4,599 occurrences of the normal `I was crazy once`. Its neighboring text still follows the same loop. Replacing the altered span with the plain sentence leaves the rest of the repetitive structure intact.

3. Preserve the altered characters exactly; replace its spaces with underscores to conform to the flag format: `cdctf{1_w@5_cr@zy_0nc3}`.

## Flag

```text
cdctf{1_w@5_cr@zy_0nc3}
```
