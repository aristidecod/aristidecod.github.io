---
title: "UnitedCTF - Vault Memo"
date: 2026-10-11 04:00:00 -0400
categories: [Writeups, Crypto]
tags: [unitedctf-2026, crypto, base64, rot13, cyberchef]
media_subpath: /assets/img/posts/vault-memo/
toc: true
---

## Context

A leaked internal memo from SelectStar Finance, "protected" by two homemade layers
that the analysts think are uncrackable. Everything is in the provided `memo.txt`.

## Solution

The file content is a block of `A–Z a–z 0–9 + / =` characters — the classic
alphabet of **Base64**. That's the outer layer. Decoding it in
[CyberChef](https://gchq.github.io/CyberChef/) with **From Base64** gives text
that's still scrambled, but in a very recognizable way: every letter shifted — a
**ROT13**.

So the recipe is just two steps, **From Base64 → ROT13**:

![CyberChef recipe: From Base64 then ROT13 revealing the memo](vm-cyberchef.png)
_From Base64 then ROT13 decodes the memo; the vault access code is the flag._

The output is the plaintext memo, with `Code: FLAG-…`.

## Takeaway

Base64 and ROT13 are **encodings, not encryption** — there's no key, so stacking
them adds zero secrecy. Anyone who recognizes the two alphabets reverses them in
seconds. "Two homemade layers" of reversible transforms is still just obfuscation.
