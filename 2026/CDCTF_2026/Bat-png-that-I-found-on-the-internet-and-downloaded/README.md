# Bat png that I found on the internet and downloaded

**Category:** Forensics

**Flag:** `cdctf{I_lik3_ba7S}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

A bat PNG and a favorite-color clue point to one image channel's least significant bits.

## Solution

1. Open the supplied bat PNG (`image-1.png`). It is a 3347 × 1233 RGBA image. The prompt singles out the bat's favorite color, **green**, which points to the green image channel rather than a generic file metadata search.

2. For each pixel, take the green value's least significant bit, `G & 1`, and render it as black or white at the same coordinates. In Pillow terms: `im = Image.open(path).convert('RGBA'); out = Image.frombytes('L', im.size, bytes(255 * (g & 1) for r,g,b,a in im.getdata()))`. The result is `green-bit0.png`.

3. Read the large handwritten characters revealed across the wings, checking the stylized digits and case against the extracted bit plane. They spell `cdctf{I_lik3_ba7S}`.

## Flag

```text
cdctf{I_lik3_ba7S}
```
