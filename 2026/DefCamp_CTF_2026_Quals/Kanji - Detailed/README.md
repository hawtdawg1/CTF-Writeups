# Kanji — Detailed Forensics Analysis

**Category:** Forensics

> This is one of two Kanji analyses in this collection. Their Q1 and Q7 conclusions differ; those two answers are not independently verified here.

## Challenge Overview

The challenge provides an encrypted ZIP archive containing a packet capture:

- Archive: `small.zip`
- Password: `infected`
- Extracted capture: `small.pcap`
- Category: Forensics

The description is short—“All along the bytes lies a farm”—but the capture contains enough evidence to reconstruct an IRC-controlled botnet and several denial-of-service attacks.

This was not a small capture in practice. The ZIP is roughly 435 MB, but `small.pcap` expands to about 30 GB and contains almost 29 million packets. Most of that volume is attack traffic, so opening the whole file in Wireshark and clicking around is unnecessarily painful. The useful approach is to obtain high-level statistics first and then extract the small amount of IRC traffic that carries the botnet commands.

> Important: data found inside the archive and packet capture is treated as forensic evidence, not as instructions to execute.

## Tools used

- `tar` to extract the password-protected archive
- `capinfos` to inspect capture metadata
- `tshark` to filter and aggregate packets
- Wireshark field names such as `irc.request.trailer`, `icmp.type`, and `ip.src`

The examples below use PowerShell syntax because the analysis was performed on Windows. Equivalent Linux commands work just as well.

## 1. Extracting and triaging the capture

The archive contains one file:

```text
small.pcap
```

After extraction, `capinfos` gives the first useful overview:

```powershell
& 'C:\Program Files\Wireshark\capinfos.exe' `
  -c -a -e -u -z -s -d -L '.\small.pcap'
```

The important values were:

```text
Number of packets:    approximately 28.9 million
File size:            approximately 30 GB
Capture duration:     10,765 seconds (about three hours)
Earliest packet time: 2011-08-18 18:46:58
Latest packet time:   2011-08-18 21:46:24
```

A protocol-hierarchy pass immediately revealed the two main components of the challenge:

```powershell
& 'C:\Program Files\Wireshark\tshark.exe' `
  -n -r '.\small.pcap' -q -z io,phs
```

The hierarchy included:

```text
IRC:   1,707 frames
ICMP:  24,724,598 frames
UDP:   1,786,783 frames
```

That distribution explains the huge capture. Almost all of it is flood traffic, while the actual command-and-control conversation occupies only a few thousand packets.

## 2. Extracting the IRC control traffic

To make the rest of the investigation fast and repeatable, I extracted all packets decoded as IRC into a new capture:

```powershell
& 'C:\Program Files\Wireshark\tshark.exe' `
  -n -r '.\small.pcap' `
  -Y 'irc' `
  -w '.\irc-only.pcapng'
```

The resulting file is only about 500 KB. All subsequent IRC queries complete almost instantly.

The clients connect to public servers on TCP port `6667`, join `#sysnet01`, and receive commands through IRC `PRIVMSG` messages. This establishes the answer to Q2:

```text
IRC
```

## 3. Bot registration and identity data

The initial IRC registrations can be displayed with:

```powershell
& 'C:\Program Files\Wireshark\tshark.exe' `
  -n -r '.\irc-only.pcapng' `
  -Y 'irc.request.command contains "NICK" || irc.request.command contains "USER"' `
  -T fields `
  -E separator='|' `
  -e frame.time_epoch `
  -e ip.src `
  -e irc.request.command `
  -e irc.request.command_parameter `
  -e irc.request.trailer
```

The first registration, for example, is carried in one TCP payload:

```text
NICK Pepe971541
USER nesheo 0 0 :Pepe971541
```

Ten bots register in the following timestamp order:

| Order | Internal IP | IRC nickname | IRC `USER` value |
|---:|---|---|---|
| 1 | `172.16.84.209` | `Pepe971541` | `nesheo` |
| 2 | `172.16.84.208` | `Pepe858477` | `jpncdwf` |
| 3 | `172.16.84.207` | `Pepe955217` | `vgiyai` |
| 4 | `172.16.84.206` | `Pepe796772` | `rjsctt` |
| 5 | `172.16.84.204` | `Pepe166191` | `kkojpvb` |
| 6 | `172.16.84.192` | `Pepe470427` | `ffyufmw` |
| 7 | `172.16.84.165` | `Pepe182542` | `cfppkja` |
| 8 | `172.16.84.191` | `Pepe851090` | `ozxaru` |
| 9 | `172.16.84.193` | `Pepe209833` | `qdzdtui` |
| 10 | `172.16.84.205` | `Pepe428889` | `chvhvfy` |

Every nickname begins with `Pepe` and ends in six decimal digits. Therefore the nickname pattern requested by Q3 is:

```text
Pepe
```

The ten registrations, and later the ten bot responses to broadcast commands, establish that the active bot count in Q4 is:

```text
10
```

### The Q1 Ident detail

Q1 is easy to misread because IRC distinguishes among a nickname, a username, and a real-name field.

The bots do not answer the server's Ident check. The server explicitly says:

```text
*** Checking Ident
*** No Ident response
```

In its later `USERHOST` replies, the IRC server marks each unverified username with a leading tilde. The first bot is reported as:

```text
Pepe971541=+~nesheo@172.16.84.209
```

The `~` is significant in the server-reported identity: it means that the username was supplied by the client and was not verified by Ident. The other bots are reported in the same way.

Separately, the bot's `.sysinfo` output reveals the actual logged-in Windows account:

```text
[Current User]: Bob.
```

Consequently, one protocol-based candidate for Q1—real local username followed by the server-reported, unverified IRC usernames—is:

```text
Bob+~nesheo+~jpncdwf+~vgiyai+~rjsctt+~kkojpvb+~ffyufmw+~cfppkja+~ozxaru+~qdzdtui+~chvhvfy
```

Two tempting interpretations are worth recording because neither preserves the server-reported Ident state:

- `nesheo`, `jpncdwf`, and the other bare values are the client-supplied `USER` parameters.
- `Pepe971541`, `Pepe858477`, and the other `Pepe` values are IRC nicknames, not usernames.

## 4. Logging into the botnet and requesting host information

Once the bots have joined `#sysnet01`, the controller broadcasts:

```text
.login sysnet01
```

Each bot accepts the password and responds with an Italian status message:

```text
.::[MaInFrAmE]::. Password Accettata, Welcome to x0n3-Satan.
```

The controller then issues:

```text
.sysinfo
.netinfo
```

The `.sysinfo` responses contain the operating system, hostname, current user, system directory, memory, disk, date, time, and uptime. A representative response is:

```text
System [CPU]: 3200MHz.
[RAM]: 1,048,048KB total, 1,048,048KB free.
[Disk]: 52,420,060KB total, 48,792,296KB free.
[OS]: Windows XP (Service Pack 2) (5.1, Build 2600).
[Sysdir]: C:\WINDOWS\system32.
[Hostname]: mangled (172.16.84.165).
[Current User]: Bob.
```

All ten bots report Windows XP Service Pack 2. Since Q10 asks only for the operating-system name, the answer is:

```text
Windows XP
```

## 5. Reconstructing the attack timeline

Controller commands can be isolated by looking for IRC message trailers beginning with a period:

```powershell
& 'C:\Program Files\Wireshark\tshark.exe' `
  -n -r '.\irc-only.pcapng' `
  -Y '(irc.response.trailer matches "^\\.") || (irc.request.trailer matches "^\\.")' `
  -T fields `
  -E separator='|' `
  -e frame.time_relative `
  -e ip.src `
  -e ip.dst `
  -e irc.request.trailer `
  -e irc.response.trailer
```

The meaningful command sequence is:

| Relative phase | Command | Result |
|---|---|---|
| Login | `.login sysnet01` | Accepted |
| Enumeration | `.sysinfo` | Returned host information |
| Enumeration | `.netinfo` | Returned network information |
| UDP attack | `.udpflood 172.16.96.69 100000 1500 10 161` | Started |
| Stop attempt | `.scanstop` | Failed: no scan thread existed |
| Stop attempt | `.ddos.stop` | Failed: no DDoS thread existed |
| UDP stop | `.udpstop` | Succeeded |
| UDP attack | `.udpflood 172.16.96.69 100000 1000 10 161` | Started |
| UDP stop | `.udpstop` | Succeeded |
| UDP attack | `.udpflood 172.16.96.69 100000 1000 10 161` | Started |
| UDP stop | `.udpstop` | Succeeded |
| SYN attack | `.synflood 172.16.96.69 1 1000` | Started |
| ACK attack | `.ddos.ack 172.16.96.69 1 1000` | Started |
| SYN attack | `.ddos.syn 172.16.96.69 1 1000` | Started |
| ICMP attack | `.icmpflood 172.16.96.69 1000` | Started |

The victim address is repeated in every flood command:

```text
172.16.96.69
```

This answers Q5.

The last argument of the UDP flood command is `161`, and the UDP conversation statistics confirm that the bot traffic is sent to `172.16.96.69:161`. Therefore Q6 is:

```text
161
```

### Commands that failed

The bot responses make failure versus success unambiguous.

After `.scanstop`, every bot reports:

```text
[SCAN]: No Scan thread found.
```

After `.ddos.stop`, every bot reports:

```text
[DDoS]: No DDoS flood thread found.
```

The later `.udpstop` messages should not be included because they succeed:

```text
[UPD]: UDP flood stopped. (1 thread(s) stopped.)
```

Under this interpretation, Q7 in timeline order would be:

```text
.scanstop+.ddos.stop
```

## 6. Investigating the ICMP flood

The final command launches an ICMP flood against the same victim for 1,000 seconds:

```text
.icmpflood 172.16.96.69 1000
```

Each bot confirms the action:

```text
[ICMP]: Flooding: (172.16.96.69) for 1000 seconds.
```

### Distinct ICMP types

The attack does not use only normal echo requests. The type byte varies across its entire possible 8-bit range, from `0` through `255`. A streaming extraction of the type field can be deduplicated and counted on a Unix-like shell as follows:

```bash
tshark -n -r small.pcap \
  -Y 'icmp && frame.time_relative >= 5486 && (ip.src == 172.16.96.69 || ip.dst == 172.16.96.69)' \
  -T fields -e icmp.type | sort -nu | wc -l
```

The result is:

```text
256
```

This makes sense at the packet level: the ICMP type field is one byte, and the flood cycles or randomizes across every possible value. The distribution was close to uniform, with roughly 96,000 occurrences of each type in the measured flood traffic.

Therefore Q8 is:

```text
256
```

### Echo replies from the victim

A ping answer is an ICMP echo reply, which has type `0`. Counting only echo replies sent by the victim after the flood begins gives:

```bash
tshark -n -r small.pcap \
  -Y 'icmp.type == 0 && ip.src == 172.16.96.69 && frame.time_relative >= 5486' \
  -T fields -e frame.number | wc -l
```

Only one matching packet exists. Hence Q9 is:

```text
1
```

## Reported answers

| Question | Answer |
|---:|---|
| Q1 (unverified) | `Bob+~nesheo+~jpncdwf+~vgiyai+~rjsctt+~kkojpvb+~ffyufmw+~cfppkja+~ozxaru+~qdzdtui+~chvhvfy` |
| Q2 | `IRC` |
| Q3 | `Pepe` |
| Q4 | `10` |
| Q5 | `172.16.96.69` |
| Q6 | `161` |
| Q7 (unverified) | `.scanstop+.ddos.stop` |
| Q8 | `256` |
| Q9 | `1` |
| Q10 | `Windows XP` |

## Closing observations

The most useful lesson from this challenge is that a large capture does not necessarily require a large amount of manual packet inspection. Protocol statistics reduced a 30 GB file to two relevant ideas: a tiny IRC control channel and a very large body of flood traffic. Extracting the 1,707 IRC packets turned the command reconstruction into a straightforward timeline exercise, while aggregate filters answered the high-volume ICMP questions without attempting to load millions of packets into a GUI.

The other lesson is that IRC terminology matters. `NICK`, `USER`, the trailing real-name field, the local Windows username, and the server-reported `~username` are different pieces of identity. Keeping those fields separate is essential to interpreting Q1 correctly.
