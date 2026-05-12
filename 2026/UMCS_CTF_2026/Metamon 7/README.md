# Metamon 7 — IOC Hashes

**Category:** DFIR
**Flag:** `UMCS{2607a3bbbc8c4482aadb9eadc13a1f40b2030f9d77be3151ed4d24a8d306fe23,f6d3de4e0827046ecd87f33906e63fb632557de53696b58c9754c8a905674aae,60b53f85e92e79e8f3894b99160bd8e65d6da9b9d4f938107939317a40b48fe5}`

## Objective

![Challenge Description](Metamon7.png)

Gather all dropped IOCs — provide file hashes in SHA256.

## Solution

Reconstructing the infection chain identifies three unique dropped tools:

| Stage | File | SHA256 |
|---|---|---|
| Persistence (UAC bypass) | `rizzler.bat` = `sentineltwo.bat` | `2607a3bbbc8c4482aadb9eadc13a1f40b2030f9d77be3151ed4d24a8d306fe23` |
| Stego delivery | `profile.png` | `f6d3de4e0827046ecd87f33906e63fb632557de53696b58c9754c8a905674aae` |
| Final payload | `Ditto.exe` | `60b53f85e92e79e8f3894b99160bd8e65d6da9b9d4f938107939317a40b48fe5` |

**Excluded from the IOC list:**
- `README.txt` — ransom note (output of attack, not attacker tooling)
- `LOCKED.7z` — attacker-created encrypted archive of victim data
- `important.txt` — victim data

## Flag

```
UMCS{2607a3bbbc8c4482aadb9eadc13a1f40b2030f9d77be3151ed4d24a8d306fe23,
f6d3de4e0827046ecd87f33906e63fb632557de53696b58c9754c8a905674aae,
60b53f85e92e79e8f3894b99160bd8e65d6da9b9d4f938107939317a40b48fe5}
```

## Full Infection Chain

```
Victim clicks fake reCAPTCHA (pinarat.github.io/a/)
    → Runs malicious PowerShell via ClickFix
        → Drops rizzler.bat (UAC bypass via fodhelper + ms-settings hijack)
            → Downloads profile.png (stego carrier)
                → Extracts & runs Stage 2 PowerShell (XOR + GZIP)
                    → Downloads and executes Ditto.exe (PyInstaller)
                        → Discord C2 (credential stealer + ransomware)
                            → Encrypts important.txt → LOCKED.7z
                            → Exfiltrates data
```

The ClickFix entry vector is actively used in the wild as of 2025–2026. DFIR work requires tracing the full kill chain — jumping straight to hashes without understanding causality would have produced wrong answers.
