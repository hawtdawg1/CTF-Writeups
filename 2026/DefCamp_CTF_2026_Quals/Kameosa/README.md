# Kameosa

**Category:** Forensics  
**Author:** Dodu Andrei  
**Difficulty:** Hard  
**Flag:** `CTF{S3a_Shant1_rul3ed_th3m_A11_7331!!!}`

## A serious safety note

The supplied archive contains a real ransomware sample and a disk image of a system on which it was executed. I treated both as hostile evidence. I did **not** execute the malware, double-click it, boot the virtual machine, or attach its disk to a running Windows guest.

All work described below was static:

- the password-protected 7-Zip archive was unpacked as data;
- the VirtualBox snapshot chain was flattened without starting the VM;
- the resulting disk was read with archive and forensic parsers;
- registry hives, event logs, Prefetch files, and encrypted documents were examined offline;
- the ransomware-encrypted ZIP files were recovered with a known-plaintext attack.

If you reproduce this investigation, keep networking, shared folders, clipboard sharing, and drag-and-drop disabled. Better still, do not boot the image at all. The password `infected` is an archive password, not an assurance that the contents are safe.

## Final answers

| Question | Answer |
|---|---|
| Q1 | `S-1-5-21-2459065227-3906730899-1249136888-1000+5` |
| Q2 | `VBOXPOST.CMD+e8f505c96b3c364e8c7951417beb9bc046e15cde` |
| Q3 | `\Device\CdRom0\VBOXPOST.CMD+\\?\D:\VBOXPOST.CMD` |
| Q4 | `malw.exe` |
| Q5 | `Bart+June-2016` |
| Q6 | `2026-07-28 19:33:24.590+2026-07-29 07:04:42.154+178` |
| Q7 | `time.windows.com+104.40.149.189:123` |
| Q8 | `VBOX HARDDISK+VBb8368025-e4856f82+98956017664` |
| Q9 | `PCI\VEN_8086&DEV_2829&SUBSYS_00000000&REV_02\3&267a616a&0&68+{939d205b-8ac3-11f1-929a-806e6f6e6963}` |
| Q10 | `9` |
| Q11 | `Microsoft-Windows-PowerShell/Operational+2026-07-28 11:15:43.634195` |
| Q12 | `.bart.zip` |
| Q13 | `WallPaper` |
| Q14 | `song.txt.txt` |
| Q15 | `CTF{S3a_Shant1_rul3ed_th3m_A11_7331!!!}` |

## Preparing the evidence

The archive contains a base VDI, a differencing snapshot, saved-state material, unattended-installation files, and VirtualBox logs. The important detail is that the latest filesystem state is not present in the base VDI alone. Looking only at the base disk would silently discard much of the incident.

I used a standalone copy of 7-Zip to inspect and unpack the outer archive:

```powershell
7z.exe l -p"infected" "DefcampMachine.7z"
7z.exe x -p"infected" "DefcampMachine.7z" -o"evidence"
```

The snapshot had the expected parent relationship to the base disk. I gave VirtualBox an isolated configuration directory and used `VBoxManage clonemedium` to merge the snapshot chain into a fresh image. This operation reads the virtual disks; it does not boot or execute their contents.

Conceptually, the two operations were:

```text
snapshot VDI + parent VDI  ->  flattened VDI
flattened VDI              ->  flattened VHD
```

The VHD conversion was simply for parser compatibility. 7-Zip can walk the NTFS filesystem inside a VHD without mounting it, which avoided exposing the host to autorun behavior or accidentally launching a file from the image.

The high-value artifacts extracted from the image were:

```text
Windows/System32/config/SAM
Windows/System32/config/SYSTEM
Windows/System32/winevt/Logs/*.evtx
Windows/Prefetch/*.pf
ProgramData/Microsoft/Windows Defender/Support/MPLog-*.log
Users/vboxuser/NTUSER.DAT
Users/vboxuser/Desktop/*
Users/vboxuser/Downloads/*
ProgramData/Microsoft/User Account Pictures/*
```

## Q1 — Interactive SID and number of local accounts

The interactive user was `vboxuser`. Its full SID appears in several artifacts, including DPAPI events:

```text
S-1-5-21-2459065227-3906730899-1249136888-1000
```

For the account count, I parsed the SAM hive and enumerated:

```text
SAM\Domains\Account\Users\Names
```

It contained five account-name subkeys:

1. `Administrator`
2. `DefaultAccount`
3. `Guest`
4. `vboxuser`
5. `WDAGUtilityAccount`

Therefore:

```text
S-1-5-21-2459065227-3906730899-1249136888-1000+5
```

## Q2 and Q3 — The setup file, Defender signature, and paths

The Windows Defender support log was unusually useful. In `MPLog-20260728-123334.log`, the first relevant scan line referenced a file on the virtual CD-ROM:

```text
\Device\CdRom0\VBOXPOST.CMD
```

The associated low-fidelity signature line contained:

```text
sigseq = 0x000042965B2654DC
sigsha = e8f505c96b3c364e8c7951417beb9bc046e15cde
```

The later Engine VFZ entry referred to the same sequence number, written without the leading zeroes. That connects the Engine VFZ entry back to this SHA-1 rather than to one of the neighboring signatures.

The same script appears elsewhere in the log through the Win32 drive-letter namespace:

```text
\\?\D:\VBOXPOST.CMD
```

The answers are consequently:

```text
VBOXPOST.CMD+e8f505c96b3c364e8c7951417beb9bc046e15cde
\Device\CdRom0\VBOXPOST.CMD+\\?\D:\VBOXPOST.CMD
```

## Q4 — Identifying the encryptor

The most direct execution artifact was:

```text
Windows\Prefetch\MALW.EXE-1F9BC0ED.pf
```

Parsing it with PECmd produced:

```text
Executable name : MALW.EXE
Run count       : 1
Last run (UTC)  : 2026-07-29 10:02:34.6423547
Source path     : \USERS\VBOXUSER\DESKTOP\MALW.EXE
```

This timestamp lands directly before the encrypted archives were created. The answer is `malw.exe`.

I never extracted or executed the binary itself. Its name and execution time were recovered from Prefetch metadata only.

## Q5 and Q12 — Malware family and encrypted extension

The ransomware wrapped each victim file in a password-protected ZIP and appended:

```text
.bart.zip
```

The ransom material, wallpaper change, and archive naming match **Bart ransomware**. Contemporary reporting places Bart's first appearance in June 2016. Two useful period references are:

- [SANS Internet Storm Center — Bart, a new ransomware](https://isc.sans.edu/diary/Bart%2Ba%2Bnew%2BRansomware/21195)
- [BleepingComputer — Bart ransomware support topic](https://www.bleepingcomputer.com/forums/t/618157/bart-ransomware-help-support-bartzip-recoverytxt/)

This gives:

```text
Bart+June-2016
.bart.zip
```

## Q6 — The dirty Security log trap

This was the easiest question to get almost right.

`Security.evtx` has its dirty flag set. The file physically contains later, recoverable records, but its EVTX header declares only **11 active chunks**. A parser that walks the declared file order gives 905 records. The first and last active records are:

```text
Record 1
2026-07-28 19:33:24.590271 UTC

Record 905
2026-07-29 07:04:42.154234 UTC
```

The challenge asks for millisecond precision, so these become `.590` and `.154`.

There are recoverable trailing chunks through record 2229, including a July 31 time-change record. Initially it is tempting to call that the last event. That produces a plausible-looking but rejected answer because those records are outside the range declared active by the dirty EVTX header.

The second trap is deciding what constitutes the encryption event. The Prefetch timestamp (`10:02:34.642`) marks process launch, but the ransomware's `.bart.zip` outputs were written at approximately:

```text
2026-07-29 10:03:11 UTC
```

Subtracting the last active Security record gives:

```text
10:03:11.000
-7:04:42.154
-------------
 2:58:28.846
```

That is **178 complete minutes** beyond the active log's coverage. The accepted answer is:

```text
2026-07-28 19:33:24.590+2026-07-29 07:04:42.154+178
```

This is a good reminder that “latest timestamp found in the file” and “last active record according to the file structure” are not always the same thing—especially after a dirty shutdown or suspended VM.

## Q7 — NTP server and contacted endpoint

The W32Time configuration in the SYSTEM hive names:

```text
time.windows.com,0x9
```

The mode suffix is configuration metadata, so the server name requested by the challenge is `time.windows.com`.

The Time Service operational log then resolves the actual network endpoint. Event IDs 264 and 265 include:

```text
time.windows.com,0x9
(ntp.m|0x9|0.0.0.0:123->104.40.149.189:123)
```

Therefore:

```text
time.windows.com+104.40.149.189:123
```

## Q8 and Q9 — Disk identity

`Microsoft-Windows-Partition/Diagnostic.evtx` contains a very complete description of disk 0 in event ID 1006. The relevant fields were:

```text
Model      : VBOX HARDDISK
Serial     : VBb8368025-e4856f82
Capacity   : 98956017664
ParentId   : PCI\VEN_8086&DEV_2829&SUBSYS_00000000&REV_02\3&267a616a&0&68
RegistryId : {939d205b-8ac3-11f1-929a-806e6f6e6963}
```

This yields:

```text
VBOX HARDDISK+VBb8368025-e4856f82+98956017664
PCI\VEN_8086&DEV_2829&SUBSYS_00000000&REV_02\3&267a616a&0&68+{939d205b-8ac3-11f1-929a-806e6f6e6963}
```

## Q10 — Successful DPAPI credential-key lookups

The relevant channel was:

```text
Microsoft-Windows-Crypto-DPAPI/Operational
```

I counted event ID `12289` entries whose message states that DPAPI found the credential key. The matching record numbers were:

```text
3, 4, 5, 7, 8, 9, 10, 11, 12
```

There are nine such events, so the answer is `9`.

## Q11 — Cleared event log

Windows records log clearing in the System channel. Event ID 104 states:

```text
The Microsoft-Windows-PowerShell/Operational log file was cleared.
```

Its precise UTC timestamp is:

```text
2026-07-28 11:15:43.634195
```

The answer is:

```text
Microsoft-Windows-PowerShell/Operational+2026-07-28 11:15:43.634195
```

## Q13 — Wallpaper registry value

The user's `NTUSER.DAT` hive contains:

```text
Key   : Control Panel\Desktop
Value : WallPaper
Data  : C:\Users\vboxuser\Desktop\recover.bmp
```

The question asks for the value **name**, not its data or the containing key, so the answer is `WallPaper`.

## Q14 — Finding the intended known plaintext

At this point the encrypted files are traditional ZipCrypto archives. Their filenames and CRCs are visible, but their contents are not. A known-plaintext attack needs a copy—or an exact reconstruction—of at least part of one encrypted file.

The challenge leaves a wonderfully human trail of hints on the desktop:

```text
SEA.txt             -> "Just a little tip here and there"
farewell and.txt    -> "Another tip"
adieu to you.txt
```

Put the filenames together with the challenge description's references to sailing and high winds:

```text
Farewell and adieu to you ... sea ...
```

That points to the traditional sea shanty **Spanish Ladies**. The encrypted desktop contains:

```text
song.txt.txt.bart.zip
```

and the entry stored inside it is named:

```text
song.txt.txt
```

The plaintext begins:

```text
Farewell an’ adieu to you fair Spanish ladies,
Farewell an’ adieu to you ladies of Spain,
For we’ve received orders for to sail for old England,
An’ hope very shortly to see you again.
```

The accepted Q14 answer is therefore `song.txt.txt`.

An unencrypted Windows account picture also forms a technically valid plaintext/ciphertext pair, and I initially used one during exploration. It can recover the same reusable ZipCrypto state, but it is not the helper file intended by the question. The desktop hints make `song.txt.txt` the canonical answer.

## Recovering the ZipCrypto keys

The recovery process is a classic known-plaintext attack:

1. Reconstruct the exact `song.txt.txt` bytes, including punctuation, UTF-8 encoding, and Windows line endings.
2. Put that plaintext into a ZIP using the same Deflate method as the encrypted archive.
3. Use `bkcrack` with the encrypted archive and the matching known-plaintext archive.
4. Recover the three internal ZipCrypto keys.
5. Apply those keys to the other `.bart.zip` files.

A representative command sequence is:

```text
bkcrack -C song.txt.txt.bart.zip -c song.txt.txt \
        -P known.zip -p song.txt.txt

bkcrack -C secret.txt.bart.zip -c secret.txt \
        -k 17999ef5 cdf44945 9df7eb9e \
        -d secret.txt
```

The recovered key triplet was:

```text
17999ef5 cdf44945 9df7eb9e
```

No brute force of the ransomware password was necessary. Once the internal state was known, the remaining archives could be decrypted directly.

## Q15 — Recovering the flag

Among the recovered files was `Desktop\secret.txt`. Its contents were refreshingly direct:

```text
Hmm you had all the luck needed in this situation and you got the flag here it is!
CTF{S3a_Shant1_rul3ed_th3m_A11_7331!!!}
```

The final flag is:

```text
CTF{S3a_Shant1_rul3ed_th3m_A11_7331!!!}
```

## Closing observations

Kameosa combines several forensic disciplines without requiring dynamic malware analysis:

- VirtualBox snapshot reconstruction;
- Windows registry analysis;
- Defender and setup-log correlation;
- EVTX structural interpretation;
- clock-change and UTC reasoning;
- Prefetch execution evidence;
- ransomware-family identification;
- and a ZipCrypto known-plaintext attack.

The two most important lessons were not really tool-specific. First, a dirty event log can contain valid-looking trailing data that is not part of its declared active range. Second, challenge prose and apparently silly files can be evidence: the sailing language was not decoration, and the tiny `SEA`, `farewell and`, and `adieu to you` files were the path to the intended plaintext.

Most importantly, none of this required running the ransomware. Static artifacts were enough to reconstruct the infection, recover the encrypted material, and obtain the flag safely.
