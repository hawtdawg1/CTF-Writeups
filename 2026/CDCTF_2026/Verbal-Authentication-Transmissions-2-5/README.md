# Verbal Authentication Transmissions 2/5

**Category:** OSINT

**Flag:** `cdctf{martin_morison}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

An audio recording uses First World War code words, and an alias list connects a rank to a name.

## Solution

1. Listen to `captured_call_LARP.mp3` and transcribe the expected-attendees portion. Two independent readings during the analysis gave **“LOWER by CHECK.”** Keep the later “will not attend” list separate; it does not negate this attendee.

2. Use the [original AEF secret code-word sheet held by the National WWI Museum](https://www.theworldwar.org/sites/default/files/2022-01/2005.102.24-secret-code-words-1.jpg). On this sheet `LOWER` denotes **Captain** and `CHECK` denotes **motorcycle**, so the caller describes the captain arriving by motorcycle.

3. Open `leaked_aliases.txt`. Its `Name - Alias` entries include `Martin Morison - Captain`. Join that record to the decoded rank, lowercase the name, replace the space with an underscore, and wrap it: `cdctf{martin_morison}`.

## Flag

```text
cdctf{martin_morison}
```
