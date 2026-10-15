# GitLash

**Category:** Misc

**Flag:** `cdctf{mY_3m41L_is_a_TOKEN_4af956e2}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

**Starting material:** the challenge prompt describes a diagnostics script pulled from local GitLab and executed as root every five minutes, and mentions that the terminal has an email tool. The instance's `player` shell exposed the repository and its README. Those artifacts connect the scheduler to the GitLab `main` branch and document an inbound email route for merge requests; the GitLab address was therefore part of the intended workflow rather than an unrelated clue.

## Solution

The approach used that workflow to change the diagnostics script, then observed the next scheduled run write a readable flag file. An initial transcription had a lowercase `l` in `3m41l` and was rejected. A byte-level read of the file showed uppercase `L` (`0x4c`) at that position.

## Flag

```text
cdctf{mY_3m41L_is_a_TOKEN_4af956e2}
```
