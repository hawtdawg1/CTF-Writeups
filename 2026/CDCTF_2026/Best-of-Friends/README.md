# Best of Friends

**Category:** Crypto

**Flag:** `cdctf{friendshipisalotlikecheese}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

A text file contains slash and backslash groups, accompanied by a Tom and Jerry clue.

## Solution

1. Open `bestfriends.txt`. It contains 26 space-separated groups made only of forward and backward slashes. The challenge invokes *Tom* and Jerry; this points to **Tomtom code**, whose symbols use `/` and `\`. Each space-delimited group represents one letter. A Morse interpretation yields gibberish because this is a different code.

2. Decode each group with a Tomtom alphabet, keeping the group boundaries. The token `/` decodes to `A`; the 26 letters read `FRIENDSHIPISALOTLIKECHEESE`, naturally spaced as **FRIENDSHIP IS A LOT LIKE CHEESE**. The 26 source groups and the 26 decoded letters provide a letter-by-letter check against the ciphertext.

3. The stated format is `cdctf{plaintextgoeshere}`. Remove the explanatory spaces and lowercase the plaintext, yielding `cdctf{friendshipisalotlikecheese}`.

## Flag

```text
cdctf{friendshipisalotlikecheese}
```
