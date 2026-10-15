# CalcuRATor 3

**Category:** Reverse Engineering

**Flag:** `cdctf{1CMP_TR1GG3R$}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

The CalcuRATor challenges inspect one ELF's process disguise and ICMP trigger behavior.

## Solution

For part 3, continue past the ICMP-type check. The binary compares the packet data to a specific 20-byte marker that decodes exactly to `cdctf{1CMP_TR1GG3R$}`. This explains why merely identifying the packet type does not activate the program's later behavior. The documented rejection concerned the **part 2** page; this part 3 flag was accepted.

## Flag

```text
cdctf{1CMP_TR1GG3R$}
```
