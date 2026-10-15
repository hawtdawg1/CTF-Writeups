# Base Alternation Trouble

**Category:** Forensics / Audio

**Flag:** `cdctf{lark pepsi shed}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

A short WAV file contains speech shifted close to the upper edge of human hearing.

## Solution

1. Open `bat.wav`. It is roughly 4.02 seconds of 48 kHz mono 32-bit floating-point audio. Ordinary playback sounds damaged because the speech energy sits mostly near a **20.219 kHz carrier**, close to the upper edge of hearing. A short-time FFT or spectrogram reveals three separate bursts around that band.

2. Shift the band down to baseband by multiplying the waveform by a 20.219 kHz sinusoid (or by taking an analytic signal and multiplying by `exp(-2πi·20219t)`), low-pass filter the result, and resample to a normal speech rate such as 16 kHz. The solve produced an audible reconstruction; the demodulated audio file `demod_20220_16k.wav` is one downshifted version. Isolate/listen to the three bursts rather than trying to read the original ultrasound-like recording directly.

3. Transcribe the bursts as `lark`, `pepsi`, `shed`. Two local speech-recognition models supported this reading; the stronger model returned “Lark Pepsi Shed” from both an alternate reconstruction and the approximately 20,219 Hz demodulation. Apply the prompt's three-word format with spaces: `cdctf{lark pepsi shed}`.

## Flag

```text
cdctf{lark pepsi shed}
```
