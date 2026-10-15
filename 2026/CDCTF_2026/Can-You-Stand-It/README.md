# Can You Stand It?

**Category:** OSINT

**Flag:** `cdctf{Geneva2009}`

**Blood award:** 3rd blood

**Verification:** Recovered from the referenced source.

## Challenge Overview

A limerick asks for the Wikipedia username of economist Guy Standing's wife and points to a dispute about his portrait.

## Solution

**Starting clue:** a limerick asks for the Wikipedia username of economist Guy Standing's wife and mentions a dispute about his picture. Search the article's Talk page and its archives for the photograph discussion, rather than guessing from biographical pages. In [Talk:Guy Standing (economist)/Archive 2](https://en.wikipedia.org/wiki/Talk:Guy_Standing_%28economist%29/Archive_2), the contributor asking for a different picture explicitly identifies herself as his wife; the linked account beside the comment is `User:Geneva2009`. Keep the username's capitalization and digits exactly as shown.

## Flag

```text
cdctf{Geneva2009}
```
