---
title: "UnitedCTF - Calcucu"
date: 2026-10-16 09:00:00 -0400
categories: [Writeups, Programming]
tags: [unitedctf-2026, programming, scripting, socket, c-semantics, integer-overflow]
media_subpath: /assets/img/posts/calcucu/
toc: true
---

## Context

Calcucu is a two-part networked speed-math challenge. Each part: connect with `nc`,
press a key to start, then answer **200 arithmetic problems** fired one after
another, each with a tight timeout (`Trop lent / Too slow` if you're late).

![start of the challenge: banner and first calculations](calcucu1-start.png)
_`Calcul N/200 : A op B = ?` — 200 of them, far too fast by hand._

No human answers 200 timed calculations, so it's a scripting problem. Both parts
share the exact same format; the only difference is **which integer type the server
computes with** — and that's where each part hides its trap.

## Calcucu 1 — C integer semantics

Answering "the obvious way" fails on negative numbers. The server uses **C integer
semantics**, which differ from Python for division and modulo:

| Operation | Server (C) | Python |
|-----------|-----------|--------|
| `-26 / 82` | **0** (truncates toward zero) | `-26 // 82` = -1 (floors) |
| `-72 % 10` | **-2** (sign of the dividend) | `-72 % 10` = 8 (sign of the divisor) |

- **Division** truncates toward zero; Python's `//` floors toward -∞.
- **Modulo** takes the sign of the left operand; Python's takes the sign of the right.

They only differ on negatives — exactly what the challenge throws at you. Reproduce
C in Python with `int(a/b)` (the `int()` truncates toward zero) and
`int(math.fmod(a, b))` (fmod gives a dividend-signed remainder).

The solver reads the banner, sends a key to start, then answers each `A op B = ?`:

```python
import socket, math

s = socket.create_connection(("challs.unitedctf.ca", 46584))
print(s.recv(4096).decode(errors="replace"), end="")   # read the banner first
s.sendall(b"2\n")                                       # then press a key to start

buf = ""
while True:
    data = s.recv(4096)
    if not data:
        break
    chunk = data.decode(errors="replace")
    print(chunk, end="", flush=True)
    buf += chunk
    while "= ?" in buf:
        before, buf = buf.split("= ?", 1)
        a, op, b = before[before.rfind(":")+1:].split()
        a, b = int(a), int(b)
        r = {"+":a+b, "-":a-b, "*":a*b, "/":int(a/b), "%":int(math.fmod(a,b))}[op]
        s.sendall(f"{r}\n".encode())
```

The 200 answers fly by and the flag drops at the end:

![all 200 solved and the flag, part 1](calcucu1-flag.png)
_After `Calcul 200/200`, the server prints the flag._

## Calcucu 2 — 8-bit char overflow

Same format and same C division/modulo, but the part 1 solver fails a few
calculations in: `-85 * 49` should be `-69`, not `-4165`.

Part 2 computes with a **signed 8-bit type** (a C `char`, range `-128..127`), so
results overflow and wrap: `-4165 mod 256 = 187`, as signed `187 - 256 = -69`. The
solver is the part 1 one with an 8-bit wrap applied to every answer:

```python
import socket, math

def wrap8(x):                     # signed 8-bit (C char): wraps into -128..127
    x %= 256
    return x - 256 if x >= 128 else x

s = socket.create_connection(("challs.unitedctf.ca", 57506))
print(s.recv(4096).decode(errors="replace"), end="")
s.sendall(b"2\n")

buf = ""
while True:
    data = s.recv(4096)
    if not data:
        break
    chunk = data.decode(errors="replace")
    print(chunk, end="", flush=True)
    buf += chunk
    while "= ?" in buf:
        before, buf = buf.split("= ?", 1)
        a, op, b = before[before.rfind(":")+1:].split()
        a, b = int(a), int(b)
        r = {"+":a+b, "-":a-b, "*":a*b, "/":int(a/b), "%":int(math.fmod(a,b))}[op]
        s.sendall(f"{wrap8(r)}\n".encode())
```

All 200 clear and the second flag drops:

![all 200 solved and the flag, part 2](calcucu2-flag.png)
_Part 2 solved with 8-bit wraparound applied to every answer._

## Takeaway

Two classic C behaviours stacked, both easy to forget from Python:

1. **Integer division/modulo of negatives is language-dependent** — C truncates
   toward zero and takes the dividend's sign; Python floors and takes the divisor's.
2. **Fixed-width integers overflow and wrap** — a `char` holds only `-128..127`, so
   any larger result silently wraps modulo 256.

When a service clearly isn't written in your language, don't assume your language's
operator behaviour or its unbounded integers — check the sign conventions and the
integer width, or you'll be wrong on every negative and every overflow.
