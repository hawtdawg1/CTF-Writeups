# Wish

**Category:** Reverse Engineering

**Flag:** `cdctf{W!sh_Up0n_A_Sh0ot1ng_Star_a18b8a1a}`

**Verification:** Displayed by the challenge instance; separate checker acceptance was not documented.

## Challenge Overview

**Starting material:** the challenge's client binary, recovered authentication code, and live terminal. The authentication structure and password storage overlap in the recovered code; the client also exposes a fixed callback address. These details explained why a valid-looking session could be reached without knowing the intended password. After authentication, the terminal provided object coordinates and a movement interface rather than displaying the flag immediately.

## Solution

The solve used the telemetry to determine that Earth was at the origin and Satellite began at `(50, 0, 0)`. An initial movement path did not trigger the objective; the corrected path produced the collision message and the flag in the terminal.

## Flag

```text
cdctf{W!sh_Up0n_A_Sh0ot1ng_Star_a18b8a1a}
```
