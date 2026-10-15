# CalcuRATor 2

**Category:** Reverse Engineering

**Flag:** `cdctf{icmp echo request}`

**Blood award:** 3rd blood (flag fix)

**Verification:** Accepted by the challenge checker.

## Challenge Overview

The CalcuRATor challenges inspect one ELF's process disguise and ICMP trigger behavior.

## Solution

For part 2, follow the only network receive path in the binary. It opens a raw IPv4 socket for ICMP and tests byte 20 of a received packet for value `8`. With the normal 20-byte IPv4 header, that byte is the ICMP type field; type 8 is an Echo Request. The checker accepted the lowercase phrase with spaces: `icmp echo request`.

## Flag

```text
cdctf{icmp echo request}
```
