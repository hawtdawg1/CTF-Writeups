# Cosmic Call

**Category:** Crypto / Network

**Flag:** `cdctf{W3_@rE_n0T_AL0n3_OuT_h3r3?_c8d45da3}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

A packet capture contains encrypted UDP messages with a reused stream cipher nonce.

## Solution

The prompt hints at a reused CTR nonce. Start with the packet capture (`cosmic_call/cosmic.pcap`) and parse its UDP payloads with a packet parser. Each application frame has a six-byte header, encrypted data, and a trailing check byte. Align the encrypted data from packets in both directions. The same stream is reused, so for any two aligned messages `C1 XOR C2 = P1 XOR P2`; known protocol words and repeated status reports reveal the **nine-byte** repeating keystream `877571f0d968f6fc`. A decoder applies that stream after stripping the frame bytes and reproduces readable messages.

Two concrete checks from the capture are packet 1, `PING SESSION=add3bb9e6b40e90a7992143ca1d5cf0`, and packet 4, a `COSMIC-1 STATUS REPORT` that ends with `ACCEPTED COMMANDS: STATUS, PING, DOWNLINK_FLAG`. These establish the session context and the flag-bearing command without guessing a flag string. The challenge session sent one `DOWNLINK_FLAG` request; the result (`cosmic_call/result.txt`) records its request and reply frames and the decrypted response, `FLAG cdctf{W3_@rE_n0T_AL0n3_OuT_h3r3?_c8d45da3}`.

## Flag

```text
cdctf{W3_@rE_n0T_AL0n3_OuT_h3r3?_c8d45da3}
```
