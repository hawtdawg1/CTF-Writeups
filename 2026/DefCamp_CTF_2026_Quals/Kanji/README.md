# Kanji

**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 136  
**Author:** Dodu Andrei  
**Status:** Q1 locked; Q7 differs from the detailed analysis

> This is one of two Kanji analyses in this collection. Their Q1 and Q7 conclusions differ; those two answers are not independently verified here.

## Overview

The challenge provides a password-protected ZIP archive (`infected`) containing `small.pcap`. The useful traffic is an IRC conversation between ten infected Windows XP machines and a controller. I followed the registrations, commands, flood traffic, and bot replies to answer Q2-Q10. Q1 remained locked after unsuccessful submissions, so the evidence and two possible interpretations are documented without claiming a solution.

## Solution

### Step 1: Find the command channel

I opened `small.pcap` in Wireshark and filtered on IRC and its server port:

```text
irc || tcp.port == 6667
```

The bots join `#sysnet01`. Frame 4513 shows the controller logging into the channel:

```text
:pepe2!~kvirc@gwsrv-27.labs.acme.io PRIVMSG #sysnet01 :.login sysnet01
```

`NICK` and `USER` registrations, channel joins, `PING`/`PONG`, and commands through `PRIVMSG` establish **Q2: `IRC`**. The controller is `pepe2`; it is not one of the bots.

### Step 2: Identify the bots

Frame 1546 contains a typical registration:

```text
NICK Pepe971541
USER nesheo 0 0 :Pepe971541
```

The bot nicknames start with `Pepe` and end in six digits. Other examples include `Pepe858477` and `Pepe955217`, giving **Q3: `Pepe`**. Ten unique bot nicknames and ten distinct internal source addresses give **Q4: `10`**:

```text
172.16.84.165  172.16.84.191  172.16.84.192  172.16.84.193
172.16.84.204  172.16.84.205  172.16.84.206  172.16.84.207
172.16.84.208  172.16.84.209
```

### Step 3: Trace the target and UDP command

The controller's UDP flood command appears in frame 8338:

```text
.udpflood 172.16.96.69 100000 1500 10 161
```

This gives **Q5: `172.16.96.69`** for the victim and **Q6: `161`** for the UDP destination port. Later flood commands use the same victim address. The UDP commands can be located with:

```text
irc.request contains ".udpflood"
```

### Step 4: Check which commands failed

I checked bot responses instead of judging the commands by name. These three TCP flood attempts returned `Done with flood (0KB/sec)`:

| Frame | Command | Bot response |
|---:|---|---|
| 4207088 | `.synflood 172.16.96.69 1 1000` | `Done with flood (0KB/sec)` |
| 4207291 | `.ddos.ack 172.16.96.69 1 1000` | `Done with flood (0KB/sec)` |
| 4207537 | `.ddos.syn 172.16.96.69 1 1000` | `Done with flood (0KB/sec)` |

In timeline order, the answer recorded here for **Q7** is **`.synflood+.ddos.ack+.ddos.syn`**. I excluded `.scanstop` and `.ddos.stop`: they are stop commands reporting that no matching thread existed.

### Step 5: Inspect the ICMP flood

At frame 4208937, the controller starts an ICMP flood. I filtered packets heading toward the victim:

```text
icmp && ip.dst == 172.16.96.69
```

The distinct `icmp.type` values cover the range 0 through 255, giving **Q8: `256`**. TShark can extract the values with:

```bash
tshark -n -r small.pcap -Y "icmp && ip.dst == 172.16.96.69" -T fields -e icmp.type
```

For the victim's echo replies (ICMP type 0), I filtered on its source IP:

```text
icmp.type == 0 && ip.src == 172.16.96.69
```

Only one packet matched, giving **Q9: `1`**.

### Step 6: Read the system reports

The bots answer `.sysinfo` with their operating system details. Searching the IRC requests for `[OS]` reveals:

```text
[OS]: Windows XP (Service Pack 2) (5.1, Build 2600)
```

Therefore **Q10: `Windows XP`**.

### Q1: An unresolved username question

Q1 asks for the bot's real reported username, followed by the fake usernames used to bypass the IRC server's mandatory identity verification, ordered by timestamp. Several fields can be confused:

- `[Current User]: Bob.` in `.sysinfo` reports the Windows username `Bob`; the period ends the sentence.
- `USER nesheo 0 0 :Pepe971541` supplies `nesheo` as an IRC username. The server later displays it as `~nesheo`; the tilde was added by the server.
- `Pepe971541` is the nickname, not the `USER` value.
- `kvirc` belongs to the controller's IRC identity.

Filtering for registrations gives the following order of first appearance:

```text
irc.request contains "NICK " && irc.request contains "USER "
```

| Frame | Bot IP | Nickname | IRC `USER` |
|---:|---|---|---|
| 1546 | `172.16.84.209` | `Pepe971541` | `nesheo` |
| 1851 | `172.16.84.208` | `Pepe858477` | `jpncdwf` |
| 2216 | `172.16.84.207` | `Pepe955217` | `vgiyai` |
| 2475 | `172.16.84.206` | `Pepe796772` | `rjsctt` |
| 2779 | `172.16.84.204` | `Pepe166191` | `kkojpvb` |
| 3045 | `172.16.84.192` | `Pepe470427` | `ffyufmw` |
| 3348 | `172.16.84.165` | `Pepe182542` | `cfppkja` |
| 3621 | `172.16.84.191` | `Pepe851090` | `ozxaru` |
| 3897 | `172.16.84.193` | `Pepe209833` | `qdzdtui` |
| 4209 | `172.16.84.205` | `Pepe428889` | `chvhvfy` |

If the question means registration order, the evidence suggests:

```text
Bob+nesheo+jpncdwf+vgiyai+rjsctt+kkojpvb+ffyufmw+cfppkja+ozxaru+qdzdtui+chvhvfy
```

But "reported" might instead refer to the order of the bots' `.sysinfo` replies. Filtering on `irc.request contains "Current User"` gives the original report frames in this order:

| Frame | Bot IP | Matching IRC `USER` |
|---:|---|---|
| 4706 | `172.16.84.165` | `cfppkja` |
| 4711 | `172.16.84.204` | `kkojpvb` |
| 4713 | `172.16.84.192` | `ffyufmw` |
| 4716 | `172.16.84.206` | `rjsctt` |
| 4723 | `172.16.84.209` | `nesheo` |
| 4725 | `172.16.84.208` | `jpncdwf` |
| 4728 | `172.16.84.193` | `qdzdtui` |
| 4766 | `172.16.84.205` | `chvhvfy` |
| 4768 | `172.16.84.207` | `vgiyai` |
| 4772 | `172.16.84.191` | `ozxaru` |

That interpretation would give:

```text
Bob+cfppkja+kkojpvb+ffyufmw+rjsctt+nesheo+jpncdwf+qdzdtui+chvhvfy+vgiyai+ozxaru
```

The platform rejected the Q1 submission and the task is now locked. Neither string is a confirmed answer; the expected order or formatting remains unresolved.

## Reported answers

| Question | Answer |
|---:|---|
| Q1 | Unresolved; see the investigation above |
| Q2 | `IRC` |
| Q3 | `Pepe` |
| Q4 | `10` |
| Q5 | `172.16.96.69` |
| Q6 | `161` |
| Q7 (unverified) | `.synflood+.ddos.ack+.ddos.syn` |
| Q8 | `256` |
| Q9 | `1` |
| Q10 | `Windows XP` |
