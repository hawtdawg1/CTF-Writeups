# Metamon 3 — Persistence Mechanism

**Category:** DFIR

**Flag:** `UMCS{Ra-T4-TA-Ta_BU7_1_doN7_w4nT_t0}`

## Objective

![Challenge Description](Metamon3.png)

What did you find after that?

## Solution

Ahmad's Temp folder (`C:\Users\ahmad\AppData\Local\Temp\`) contains two batch files: `rizzler.bat` and `sentineltwo.bat`. They are byte-for-byte identical.

The batch file's logic:

1. Copies itself.
2. Registers a custom URL handler under `HKCU\Software\Classes\osIrizz\Shell\Open\command`.
3. Points `HKCU\Software\Classes\ms-settings\CurVer` to that handler.
4. Launches `fodhelper.exe` — a UAC auto-elevating binary — which executes the registered handler without a UAC prompt.
5. Writes the flag string directly into `HKCU\Software\Classes\ms-settings\CurVer`.

Reading the batch file reveals the flag in the registry write step.

## Flag

```
UMCS{Ra-T4-TA-Ta_BU7_1_doN7_w4nT_t0}
```

The `fodhelper.exe` UAC bypass via `ms-settings` registry hijack is a well-known Living-off-the-Land technique still actively used in commodity malware.
