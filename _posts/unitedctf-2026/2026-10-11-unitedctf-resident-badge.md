---
title: "UnitedCTF - Resident Badge"
date: 2026-10-11 03:00:00 -0400
categories: [Writeups, Reverse]
tags: [unitedctf-2026, reverse, ghidra, xor, strings]
media_subpath: /assets/img/posts/resident-badge/
toc: true
---

## Context

We recovered the firmware of SelectStar's building badge reader. It asks for an
access code. The description offers two routes: *clone a valid badge, or rip the
flag straight from the binary.* We'll do both.

## Recon

The file is a 64-bit ELF, **not stripped**:

![file badge output](rb-file.png)
_64-bit ELF, dynamically linked, not stripped — symbols are intact._

Running it asks for a badge access code and rejects a wrong guess:

![running the badge, wrong code refused](rb-run-refused.png)
_"Entrez le code d'acces du badge" — a bad code raises a security alert._

`strings` on the binary surfaces something that looks like the password —
`ZSTARFALL-2026`:

![strings output with ZSTARFALL-2026](rb-strings.png)
_A promising string… with a leading `Z`._

But entering `ZSTARFALL-2026` is refused:

![ZSTARFALL-2026 rejected](rb-wrong-code.png)
_The `strings` candidate doesn't work — the leading `Z` is noise from an adjacent string._

## Ghidra

Loading the binary in Ghidra, `main` shows the check: it reads the code, strips the
newline, and `strcmp`s it against `EXPECTED`; on a match it calls `reveal()`:

![Ghidra decompilation of main](rb-ghidra-main.png)
_`main`: `strcmp(input, EXPECTED) == 0` → `reveal()`._

Looking at `EXPECTED` in the disassembly, the real code is **`STARFALL-2026`** —
no leading `Z` (that byte belonged to a neighbouring string in the `strings`
dump):

![EXPECTED = STARFALL-2026](rb-expected.png)
_`EXPECTED` resolves to `"STARFALL-2026"`._

`reveal()` is where the flag lives. It isn't stored in clear — it's an `enc` array
XOR-decoded one byte at a time with `0x5a`, then printed:

![Ghidra decompilation of reveal](rb-ghidra-reveal.png)
_`reveal()`: `local_98[i] = enc[i] ^ 0x5a` over 37 bytes, then `printf("… %s", local_98)`._

## Solution 1 — clone the badge

The simplest route: just enter the correct code. `STARFALL-2026` passes the
`strcmp`, `reveal()` runs, and the program prints the decoded flag itself.

## Solution 2 — rip it from the binary

No need to run anything: the flag is `enc XOR 0x5a`. Note that `local_98` is just
the empty stack buffer the decoded flag is written to at runtime — the data we want
is the **`enc`** array. Double-clicking `enc` in the `reveal` decompiler jumps to
its 37 bytes in the data section (here at `0x404080`):

![the enc array bytes in Ghidra](rb-enc-array.png)
_The 37 encoded bytes of `enc` — XOR each with `0x5a` to recover the flag._

Dump those bytes and XOR each with `0x5a`, for example in CyberChef (**XOR** key
`5a` hex) or a one-liner:

```python
enc = bytes([0x1c,0x16,0x1b,0x1d,0x77, ... ])   # the 37 bytes from Ghidra
print(bytes(b ^ 0x5a for b in enc).decode())
```

Both routes give the same flag.

## Takeaway

A hardcoded secret is recoverable whether it's in clear or lightly obfuscated. XOR
with a constant is not encryption — the key is right there in the code (`0x5a`).
And `strings` is a first pass, not ground truth: it happily glues adjacent bytes
together, which is why the `Z` led to a dead end until Ghidra showed the real
`EXPECTED`.
