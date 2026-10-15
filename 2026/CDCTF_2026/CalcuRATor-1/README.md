# CalcuRATor 1

**Category:** Reverse Engineering

**Flag:** `cdctf{wpad}`

**Blood award:** 3rd blood

**Verification:** Derived from static analysis of the supplied binary.

## Challenge Overview

The CalcuRATor challenges inspect one ELF's process disguise and ICMP trigger behavior.

## Solution

Begin with the supplied `calculator` ELF. The first prompt asks what the malicious component calls itself after starting. Its startup path checks for root, starts a new session, and copies the literal string `wpad` over `argv[0]`, changing the name shown in process listings.

## Flag

```text
cdctf{wpad}
```
