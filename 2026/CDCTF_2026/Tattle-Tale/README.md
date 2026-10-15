# Tattle Tale

**Category:** Misc

**Flag:** `cdctf{D0wnTh3Da1sEeCha1n!_ca47be69}`

**Blood award:** 1st blood

**Verification:** Displayed by the challenge instance.

## Challenge Overview

**Starting material:** the prompt's Alice-to-Bob-to-Carol story, a live terminal, and the terminal's briefing and SSH material. The challenge asks the participant to get back to Alice's machine. The briefing establishes a numbered host chain and a periodic job that runs a script supplied on the preceding host. That schedule explains why an ordinary file search on the starting shell could not produce the flag: the relevant data was at the far end of the relay.

## Solution

The solve followed the chain in order and discovered that it continued beyond the four hosts initially named in the briefing. The terminal eventually returned `cdctf{D0wnTh3Da1sEeCha1n!_ca47be69}` from the target side. The solver checked the returned bytes rather than relying only on the rendered terminal line.

## Flag

```text
cdctf{D0wnTh3Da1sEeCha1n!_ca47be69}
```
