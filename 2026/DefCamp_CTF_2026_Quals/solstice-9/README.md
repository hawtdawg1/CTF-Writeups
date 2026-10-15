# solstice-9

**Category:** Misc  
**Difficulty:** Hard  
**Points:** 140  
**Date:** 2026-09-19  
**Author:** Synthz  
**Flag:** `DCTF{c7da32e39bd5727769742639fb9855971cb88cb952573182913b9155d7b1b286}`

## Challenge Overview

The challenge supplied a Raspberry Pi firmware image and two bench captures from a retired solar controller. The controller's `powerd` service exposed an undocumented maintenance path protected by an Ed25519 challenge-response. The signing material was not stored in one place: a calibration fixture was divided among an SPI flash journal, an I2C EEPROM capture, and an inverted-UART supervisor capture.

## Solution

### 1. Inspect the firmware

The image contained FAT boot and ext4 root partitions. I examined the root partition read-only and found the stripped `powerd` service, the `boarddiag` utility, and the archived board captures. A local token file held a decoy flag, so reading the firmware alone did not solve the remote challenge.

### 2. Trace the maintenance flow

Disassembly showed that the undocumented maintenance path issues a short-lived nonce and accepts a signature over it. The service checks that signature against a public key embedded in `powerd`. The companion `boarddiag` utility derives the corresponding signing identity from a calibration fixture.

### 3. Reconstruct the fixture

The fixture had three 32-byte components:

1. **SPI journal:** de-whiten the flash pages, validate the committed records by checksum, and select the newest generation for the controller revision.
2. **I2C EEPROM:** decode the bus capture and recover the revision-specific calibration block.
3. **UART supervisor:** invert the captured line, decode the serial frames, and select the valid revision packet by checksum. Another packet with the same revision was a checksum trap.

Combining those components and checking the result with `boarddiag` produced a signing identity whose public key matched the one in `powerd`.

### 4. Obtain the receipt

The validated signing identity completed the maintenance challenge-response, and the service returned the flag in its receipt. The derived signing material and live service endpoint are omitted from this public writeup.

## Flag

```text
DCTF{c7da32e39bd5727769742639fb9855971cb88cb952573182913b9155d7b1b286}
```
