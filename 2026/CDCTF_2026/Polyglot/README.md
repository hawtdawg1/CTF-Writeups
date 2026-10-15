# Polyglot

**Category:** Reverse Engineering

**Flag:** `cdctf{SecretOfComputerLanguages}`

**Blood award:** 1st blood (flag fix)

**Verification:** Full answer supplied separately; the attached files confirm its prefix and length but do not uniquely determine its suffix.

## Challenge Overview

**Starting material:** `Polyglot1.zip`, which contains `Program1.hs`, `Questions1.txt`, and `Answers1.txt`. The source imports a missing `Secrets.KEY`; the job is to infer that key from the 32 question/answer lines. The prompt gives the format `cdctf{ABunchOfTitleCaseWords}`.

## Solution

In `Program1.hs`, six parallel locale states begin at 1, 2, 3, 4, 5, and 6. For each character with code point `c`, a state `n` selects a form using `(n*c) mod 6`; the next state is `(n*c) mod 7`. The program converts those forms into vocabulary such as `You`/`Yous`, `Here`/`Heres`, and `This`/`These`. Each question requests three of the six forms, and the corresponding answer line reveals their indices. Work through each line by trying characters permitted by the Haskell bounds (`'A' <= c <= '}'`) and retaining characters whose three selected forms match the answer. This reconstructs the beginning `cdctf{SecretOfComp`.

There is a limit to what the supplied question and answer files prove. Its final `p` has ASCII value 112, divisible by 7. Consequently all six next states become zero. Zero remains zero for every later character, so the remaining answer lines are always the Alpha forms, regardless of the suffix. The supplied files therefore verify the prefix and total length, but they cannot distinguish the rest of the key. The full answer is `cdctf{SecretOfComputerLanguages}`. An independent simulator of the equations above reproduced all 32 question/answer lines with that key, but other suffixes also do so; the suffix cannot be recovered uniquely from the ZIP alone.

## Flag

```text
cdctf{SecretOfComputerLanguages}
```
