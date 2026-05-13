# Shadow_Ops

**Category:** Forensics

**Flag:** `umcs{dns_fragmentation_shadow_ops}`

## Challenge Overview

> *"Unusual DNS activity was observed from an internal host prior to an employee's departure. The traffic appears routine at first glance, but something doesn't add up. Investigate the capture and recover the hidden message."*

One file: `Shadow_Ops.pcap`

## Initial Enumeration / Recon

```bash
capinfos Shadow_Ops.pcap
# 388 packets, all DNS, Raw IPv4
```

All 388 packets are DNS queries. Extracting all unique DNS query names reveals four domain families:

| Domain Family | Role |
|---|---|
| `tracking-data.net` | Base64-encoded garbage strings — noise |
| `analytics-cdn.net` | 8-character hex subdomains — noise |
| `update-service.net` | `prefix-NNNNN` format — noise |
| `metrics-update.net` | 10 short alphanumeric subdomains — **signal** |

The first three families are decoys. The real data is hiding in `metrics-update.net`.

## Vulnerability Analysis

Examining the DNS **responses** reveals something unusual: while most records have a TTL of 300, the `metrics-update.net` responses have TTLs ranging from 101 to 110.

```bash
tshark -r Shadow_Ops.pcap -Y "dns.flags.response == 1" \
    -T fields -e dns.resp.name -e dns.resp.ttl \
    | grep "metrics-update" | sort -k2 -n
```

The TTL field is being repurposed as a **sequence number**. The last entry `aaaaa=` ends with `=` — base32 padding. The 10 subdomains, sorted by TTL, form a base32-encoded payload.

## Exploitation Steps

```python
import base64, gzip

chunks = [
    "d6fqqcdnbs",  # TTL 101
    "2gsaadmrqx",  # TTL 102
    "iyiabp2xkd",  # TTL 103
    "voj3esxdsp",  # TTL 104
    "fnfezt2nzu",  # TTL 105
    "vuslgjzthy",  # TTL 106
    "wl6ojbgmsl",  # TTL 107
    "4pz4xsrlqf",  # TTL 108
    "achwodurei",  # TTL 109
    "aaaaa=",      # TTL 110
]

combined = "".join(chunks).upper()
decoded  = base64.b32decode(combined)

result = gzip.decompress(decoded)
print(result)
# b'UMCS{dns_fragmentation_shadow_ops}'
```

## Flag

```
umcs{dns_fragmentation_shadow_ops}
```

DNS exfiltration via subdomain labels is a real technique used by actual malware (DNScat, Cobalt Strike DNS beacons). The clever part here was using the TTL field as a steganographic sequence number — something you'd likely miss if you only looked at query names. Always check response attributes (TTL, RDATA), not just query strings.
