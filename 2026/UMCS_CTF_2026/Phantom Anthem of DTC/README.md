# Phantom Anthem of DTC

**Category:** Crypto

**Flag:** `UMCS{ILMU_PUNCA_KEMAJUAN}`

## Challenge Overview

> *"Dewan Tunku Canselor (DTC) has witnessed countless graduations, its halls forever echoing with the university anthem. We recently intercepted this year's test broadcast… Yet, rumors suggest a phantom lurks within the soundwaves, waiting for the perfect moment to speak a sacred truth."*
>
> *"Remember: In the realm of acoustics, true silence is only achieved when opposites collide. Only from that silence can the ghost be awakened."*
>
> **Hint:** *"Focus on the physical properties of stereo sound — what happens when two identical but opposing forces meet? As for the encrypted message you uncover, the father of our university holds the 'key', but you'll need to know exactly when he took his oath to find the right one."*
>
> **Note:** Convert your final decrypted text to UPPERCASE and connect the words with underscores. Format is `UMCS{WORD1_WORD2_WORD3}`.

A WAV audio file was provided.

## Exploitation Steps

**Stage 1 — Extract the Side channel:**

The phrase "true silence is only achieved when opposites collide" describes **Mid-Side (M/S) audio processing**. When L and R channels are identical, subtracting them reveals hidden content.

```python
import numpy as np
import wave, sys

def extract_side_channel(input_file, output_file="side_channel.wav"):
    with wave.open(input_file, 'rb') as w:
        channels  = w.getnchannels()
        sampwidth = w.getsampwidth()
        framerate = w.getframerate()
        frames    = w.readframes(w.getnframes())

    dtype   = np.int16 if sampwidth == 2 else np.int32
    samples = np.frombuffer(frames, dtype=dtype).reshape(-1, 2)
    L = samples[:, 0].astype(np.int32)
    R = samples[:, 1].astype(np.int32)
    side = np.clip(L - R, -32768, 32767).astype(np.int16)

    with wave.open(output_file, 'wb') as out:
        out.setnchannels(1)
        out.setsampwidth(2)
        out.setframerate(framerate)
        out.writeframes(side.tobytes())

extract_side_channel("anthem.wav")
```

Playing `side_channel.wav` reveals a clear Morse code signal.

**Stage 2 — Decode the Morse code:**

```
LUYKB BFZEJNHPKEXGNTOTH IUYCO
```

**Stage 3 — Vigenère decryption:**

The hint says *"the father of our university holds the 'key'"*. Universiti Malaya's founding father is **Tunku Abdul Rahman** — key: `TUNKU`.

Applying Vigenère decryption (P = (C − K) mod 26):

| Ciphertext | Key | Plaintext |
|---|---|---|
| `LUYKB` | `TUNKU` | `SALAH` |
| `BFZEJNHPKEXGNTOTH` | `TUNKUTUNKUTUNKUT` | `ILMUPUNCAKEMAJUAN` |
| `IUYCO` | `TUNKU` | `PALSU` |

Full phrase: **SALAH ILMU PUNCA KEMAJUAN PALSU**

**Stage 4 — Pick the right three words:**

`SALAH` and `PALSU` ("wrong/false") are misdirection labelled by the challenge itself. The three middle words form Universiti Malaya's actual motto:

> **ILMU PUNCA KEMAJUAN** — *"Knowledge is the Source of Progress"*

## Flag

```
UMCS{ILMU_PUNCA_KEMAJUAN}
```

This challenge stacked multiple steps elegantly. The M/S stereo trick is genuinely used in audio engineering. The Vigenère decryption was straightforward once the key hint clicked. The misdirection words in the output required thinking about context rather than just running a decoder.
