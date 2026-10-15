# Verbal Authentication Transmissions 5/5: You get all that?

**Category:** Crypto

**Flag:** `cdctf{8u88l3_848813_fl4g_pa55ing}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

A call transcript and recording provide 17 five-letter code groups to decode.

## Solution

1. Read `cred_call_transcript.txt` alongside `captured_cred_call.mp3`. The transcript supplies 17 five-letter groups that are difficult to hear reliably: `ximok gemul ganol ruvul hevaf murof felyf metuf myvaf cusih zynok sitek lulol bemef hatok norek lizyx`.

2. The `x` boundary and alternating pronounceable consonant/vowel pattern identify Bubble Babble. Decode the groups as one continuous Bubble Babble stream, preserving their order. Each group encodes one or two bytes with a running seed/checksum; treating the groups as separate words loses that state. A Bubble Babble decoder tracks the running seed and checks each group.

3. The script prints `words 17`, `bytes 33`, and `decoded cdctf{8u88l3_848813_fl4g_pa55ing}`. It then re-encodes those 33 bytes and reports `roundtrip True`, matching every one of the 17 original groups.

## Flag

```text
cdctf{8u88l3_848813_fl4g_pa55ing}
```
