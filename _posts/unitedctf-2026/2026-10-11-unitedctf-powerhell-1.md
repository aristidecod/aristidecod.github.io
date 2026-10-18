---
title: "UnitedCTF - PowerHell 1"
date: 2026-10-11 07:00:00 -0400
categories: [Writeups, Misc]
tags: [unitedctf-2026, misc, powershell, jail, sandbox-escape]
media_subpath: /assets/img/posts/powerhell/
toc: true
---

## Context

PowerHell 1 is a PowerShell jail exposed over `nc`. It prints its own source, a
prompt `PH >`, and dares us to *escape the (s)hell*. Whatever we type is eventually
run with `iex` (Invoke-Expression) — but only after passing a blacklist filter.

## Recon

On connect, the service shows the loop that gates our input:

![the jail source and its VerifyInput filter](ph1-jail-code.png)
_The read loop builds a blacklist, checks input against it, then runs `iex $in`._

The blacklist `$bl` is built from the live environment:

- `(Get-Command).Name` — every command name
- `(Get-Command -Verb Get).Noun` — every `Get-*` noun (`Content`, `ChildItem`, `Item`…)
- `(Get-Alias).Name` — every alias (`gc`, `cat`, `ls`, `dir`…)
- `(Get-Verb).Verb` — every verb (`Get`, `Set`, `New`…)
- the characters `[`, `]`, the backtick, and the string `bl`

## Exploitation

The weakness is in that line: the check is a **literal substring** match
(`IndexOf`). It only sees a forbidden word if it appears *contiguously* in our
input. So we rebuild command names by **concatenation** — the blacklisted token
never appears whole — and invoke the result with the call operator `&`, which
isn't filtered (nor are `"`, `+`, `(`, `)`).

List the directory by reconstructing `gci`:

```powershell
&("g"+"ci")
```

That reveals `flag.txt` in `/app`. Reading it needs `Get-Content`; its alias `cat`
turns out not to be registered here, but `gc` works — both are rebuilt the same
way, and the filename `flag.txt` is split too (plain `flag` is caught):

```powershell
&("g"+"c") ("f"+"lag.txt")
```

![gci listing then gc reading the flag](ph1-exploit.png)
_`&("g"+"ci")` lists `/app`; `&("g"+"c") ("f"+"lag.txt")` prints the flag._

In `"g"+"c"` and `"f"+"lag.txt"` the real characters are `g"+"c` and `f"+"lag.txt`
— so `gc`, `Get`, `Content` and `flag` never appear contiguously, and the filter
waves them through.

## Takeaway

A denylist built from literal substrings is trivially bypassed: if the language
can build a string at runtime (`+`) and call it (`&`, `iex`), the forbidden word
need never be typed as one piece. Sandboxing an interpreter means restricting
*capabilities* (constrained language mode, no `iex`), not blacklisting *words*.
