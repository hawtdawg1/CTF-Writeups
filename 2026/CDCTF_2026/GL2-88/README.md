# GL2-88

**Category:** Crypto / Reverse Engineering

**Flag:** `cdctf{TooConfusedlyDiffusedAcrossMyGroup???}`

**Blood award:** 1st blood

**Verification:** Recovered or checked against supplied material.

## Challenge Overview

**Starting material:** the ciphertext below from the prompt and `GL2-88.hs` from the download. The Haskell source specifies a 67-character alphabet, mapping `<` to 0 and `~` to 66, and an 11-character key. Read `encrypt`, `getKeyStream`, and the two matrices `_A` and `_B` before attempting to crack the text.

## Solution

```text
XwFAN`aXUspHB~]bhjV>_Fmk}VJ~tx=BwsvP<hK[XOo`
```

The apparent matrix cipher is linear in the key characters. `m0` consists of upper-triangular matrices whose upper-right entries are the key characters mapped into `Z67`. The group operations and the twelve `round` applications preserve that form. At each ciphertext position, multiplication by the plaintext vector `(c2s(character), -1)` gives an affine expression modulo 67 in the eleven key values. The known `cdctf{` prefix and final `}` provide seven equations. They leave four independent key values to recover.

One reproducible way to build the equations is to evaluate the Haskell `encrypt` function with the all-`<` key (all eleven key scalars zero), then repeat with one `<` replaced by `=` at each key position (one scalar set to one). Subtract the baseline ciphertext values modulo 67 to get each key coefficient at each known-plaintext position. Solve the resulting linear equations over `Z67`; four free field values remain, giving `67^4 = 20,151,121` possibilities. Generate the associated plaintexts and rank those with the flag prefix and brace by readable English words and the prompt's diffusion/group theme. This yields the key `LoveNilsAbe` and plaintext `cdctf{TooConfusedlyDiffusedAcrossMyGroup???}`.

Check the result with the original program: `runghc GL2-88.hs decrypt LoveNilsAbe '<ciphertext>'` gives the flag, and `runghc GL2-88.hs encrypt LoveNilsAbe 'cdctf{TooConfusedlyDiffusedAcrossMyGroup???}'` reproduces the prompt's ciphertext byte for byte.

## Flag

```text
cdctf{TooConfusedlyDiffusedAcrossMyGroup???}
```
