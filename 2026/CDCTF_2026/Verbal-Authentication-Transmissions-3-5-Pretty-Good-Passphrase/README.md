# Verbal Authentication Transmissions 3/5: Pretty Good Passphrase

**Category:** Crypto

**Flag:** `cdctf{pr3t7y_g00d_piv4cy_fl4G}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

A voicemail conveys an encrypted message through spoken PGP word-list terms. A private key and passphrase are supplied.

## Solution

1. The supplied files are the long `voicemail.mp3` and the armored private key `VAT_key`; the challenge also supplied the key's passphrase. The voicemail is about 11 minutes of individual code words. Identify the PGP word-list scheme: each byte is spoken as a word, alternating the **even** and **odd** columns according to byte position. A PGP word-list lookup table maps the spoken words back to bytes.

2. Transcribe one word per position, then map each word through the appropriate parity column to a byte. The first bytes form the literal ASCII header `-----BEGIN PGP MESSAGE-----`; that known header helps confirm the alignment. Decode the remainder into the complete armored PGP message, including its base64 body and `=` checksum line.

3. Speech recognition can confuse similar words. Check doubtful positions against the audio and the alternating-column vocabulary. The analysis also compared repeated word waveforms and used the PGP packet structure to resolve words in the encrypted session-key packet. Validate the armored message's CRC24 checksum before decryption; a valid checksum detects any remaining transcription error.

4. Import `VAT_key` into a temporary local GPG home and decrypt the reconstructed armored message with the supplied passphrase. In the solve, GPG exited successfully and produced `Good work: cdctf{pr3t7y_g00d_piv4cy_fl4G}`. Extract only the flag from that plaintext. The GPG success and matching armor checksum verify the transcription.

## Flag

```text
cdctf{pr3t7y_g00d_piv4cy_fl4G}
```
