---
title: "UnitedCTF - QuickBag"
date: 2026-10-11 11:00:00 -0400
categories: [Writeups, Web]
tags: [unitedctf-2026, web, cookie, user-agent, sql-injection, barcode, burp]
media_subpath: /assets/img/posts/quickbag/
toc: true
---

## Context

QuickBag is a hastily-built grocery app where products keep appearing out of
nowhere. The series has three flags, each hiding in a different trusted-but-
controllable input: a cookie, the `User-Agent`, and a scanned barcode.

## QuickBag 1 — the flag is the cookie

The first flag needs no attack at all. Opening the browser devtools on the
**Application → Cookies** tab, the session carries a cookie named `tag` whose
value is the flag:

![flag stored in the tag cookie](qb1-cookie.png)
_The `tag` cookie holds `FLAG-…` directly._

## QuickBag 2 — spoofing the Zebra scanner

The scanning feature refuses ordinary browsers:

![scanner authorization required](qb2-scanner-required.png)
_"Barcodes can only be scanned from a Zebra handheld barcode scanner device."_

The server decides whether we're a Zebra device from the one thing we fully
control: the `User-Agent` header. Intercepting the request to `/scan` in Burp and
swapping in a Zebra scanner UA is enough:

```
User-Agent: Mozilla/5.0 (Linux; Android 11; TC21 Build/R02-02) AppleWebKit/537.36 (KHTML, like Gecko) Zebra/1.0 EnterpriseBrowser/3.0
```

![Burp request with the Zebra User-Agent](qb2-useragent.png)
_Rewriting the `User-Agent` to a Zebra EnterpriseBrowser string._

The scanner terminal loads, and the second flag sits in the page header:

![scan page response containing the flag](qb2-flag.png)
_"Zebra scanner terminal - FLAG-…" in the response._

## QuickBag 3 — SQL injection through a barcode

With the scanner unlocked, we can submit barcodes. The decoded barcode value is
looked up against the product database — an input channel that still ends up in a
SQL query. So instead of a real product code, we encode a SQL injection payload
into a barcode.

A short script generates Code128 barcodes for a list of payloads:

```python
import barcode
from barcode.writer import ImageWriter

OPTS = {"module_width": 0.6, "module_height": 30.0, "quiet_zone": 8.0, "font_size": 10}
barcode.Code128("' OR 1=1-- -", writer=ImageWriter()).save("bc_ortrue", options=OPTS)
```

![Code128 barcode encoding ' OR 1=1-- -](qb3-barcode-ortrue.png)
_A Code128 barcode that decodes to `' OR 1=1-- -`._

The classic `' OR 1=1-- -` turns the lookup into an always-true condition, so the
query returns **every** product — including the hidden flag product (price
`$0.00`, stock `0`):

![scan result dumping all products including the FLAG product](qb3-scan-result.png)
_`' OR 1=1-- -` dumps the full product list; the last entry is the flag._

## Takeaway

All three flags come from the server trusting a value the client owns: a cookie, a
request header, and the contents of a barcode. The barcode is the interesting one —
a physical-looking input channel is still just a string reaching a SQL query, and
must be parameterized like any other user input.
