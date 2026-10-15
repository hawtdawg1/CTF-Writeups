# Ghost in the Pool

**Category:** Forensics

**Flag:** `cdctf{YVR_NRT_HND_SEA}`

**Verification:** Accepted as the challenge flag.

## Challenge Overview

**Starting material:** `pineap-capture.db`, a SQLite database whose `log` table has `mac`, `ssid`, `created_at`, and `updated_at` fields. The prompt says one device leaked its saved-network history instead of randomizing its MAC and asks for four airport codes in collection order.

## Solution

First inspect the schema with `sqlite3 pineap-capture.db '.schema'`. Group probes by MAC, then inspect devices with multiple distinct SSIDs. A locally administered address has bit `0x02` set in its first octet; the relevant `3C:07:54:1D:9B:E2` has that bit clear. It has a coherent nine-network travel trail, including four plainly labeled airport SSIDs. Deduplicate repeated probes by first sighting and order by `MIN(created_at)`:

```sql
SELECT ssid, MIN(created_at) AS first_seen
FROM log
WHERE mac = '3C:07:54:1D:9B:E2'
GROUP BY ssid
ORDER BY first_seen;
```

The airport entries appear as `YVR-Public`, `NRT-FREE-WiFi`, `HND-FreeWiFi`, and `SEA-WiFi-Free`, in that order, with other residential, hotel, and inflight networks interleaved. Keep only the airport codes for the required `AAA_BBB_CCC_DDD` format.

## Flag

```text
cdctf{YVR_NRT_HND_SEA}
```
