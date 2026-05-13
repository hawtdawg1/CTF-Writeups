# Varsity Lake Sonata

**Category:** Reverse Engineering

**Flag:** `UMCS{t4sik_v4rs1ty_pi4n0_x0r_m3l0dy}`

## Challenge Overview

![Challenge Description](VarsityLakeSonata.png)

> A single ELF64 binary named `varsity_lake_sonata` was provided.

## Initial Enumeration / Recon

```bash
file varsity_lake_sonata
# ELF 64-bit LSB executable, x86-64, dynamically linked, stripped

detect-it-easy varsity_lake_sonata
# Packer: UPX(4.24)
```

DiE identifies it as UPX-packed, but the standard unpacker fails:

```bash
upx -d varsity_lake_sonata
# upx: varsity_lake_sonata: NotPackedException: not packed by UPX
```

Looking at the binary in a hex editor, the regions where UPX normally places its magic bytes (`UPX!`) have been replaced with custom values — `0x00ec`, `0x078d`, `0x0f9a`, `0x1908`, and `0x1910`. The UPX header has been deliberately corrupted to stop the standard unpacker.

## Exploitation Steps

**Step 1 — Let the binary unpack itself in memory:**

Run the binary and leave it waiting for input, then dump its memory from a second terminal:

```python
# Parse /proc/<pid>/maps, find the executable r-xp segment
# Read that region from /proc/<pid>/mem
# Write to unpacked.elf
```

**Step 2 — Analyze the unpacked binary in IDA:**

The check function contains a XOR loop with hardcoded 64-bit constants:

```c
v2[0] = 0xD0B47615E415408LL;
v2[1] = 0x4206036F73465FLL;
v2[2] = 0x40105407795118LL;
v2[3] = 0xD00627C141F733ALL;
v3 = 1785070305;
v5 = 19;

for (i = 0; i <= 35; i++) {
    if ((v5 ^ i ^ (char)(i + a1) ^ 0x4D) != (unsigned __int8)v2[i])
        return 0;
    v5 = *(BYTE*)(i + a1);
}
return 1;
```

The input is XORed character-by-character with `v5` (previous input byte), `i`, and `0x4D`. Inverting this logic yields the flag directly.

## Flag

```
UMCS{t4sik_v4rs1ty_pi4n0_x0r_m3l0dy}
```

Tampered UPX headers are a common anti-analysis trick. Running a binary and pulling its memory pages mid-execution is a reliable fallback when static unpacking isn't possible.
