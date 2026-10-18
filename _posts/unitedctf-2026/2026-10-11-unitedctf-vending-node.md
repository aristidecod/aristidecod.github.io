---
title: "UnitedCTF - Vending Node"
date: 2026-10-11 09:00:00 -0400
categories: [Writeups, Pwn]
tags: [unitedctf-2026, pwn, buffer-overflow, stack]
media_subpath: /assets/img/posts/vending-node/
toc: true
---

## Context

Vending machine #42 only dispenses items if you pay — unless you enable its "VIP"
mode reserved for management. The description spells out the bug: *the code never
checks the size of what you type.* Get the item for free.

## Recon

The binary is a 64-bit ELF, not stripped. Checking the protections with
`checksec` shows **no stack canary** and **no PIE**:

![checksec output](vn-checksec.png)
_No canary and No PIE — nothing between the buffer and its neighbours._

Disassembling `main` in pwndbg shows the whole story:

![pwndbg disassembly of main](vn-pwndbg-main.png)
_`main`: `vip` at `rbp-0x10` is zeroed, the article code is read into the buffer at `rbp-0x30`._

Local variables live at fixed offsets from `rbp` (the base of the stack frame).
Here the buffer and the VIP flag are neighbours on the stack:

```
rbp-0x30  ┌─────────────────────────┐
          │  article buffer         │  ← read() writes here
          │  (32 bytes)             │
rbp-0x10  ├─────────────────────────┤
          │  VIP flag  = 0          │  ← sits right after the buffer
          └─────────────────────────┘
```

`0x30 - 0x10 = 0x20` = **32 bytes** between the start of the buffer and the VIP
flag. Since `read` doesn't limit the length, anything beyond 32 bytes spills out
of the buffer and overwrites the VIP flag. Right after, `main` reloads `rbp-0x10`;
if it's no longer `0`, it calls `win()`, which reads `/flag.txt`.

So typing 40 `1`s fills the 32-byte buffer and writes the extra 8 over the VIP flag
(now non-zero) — VIP mode on. We never touch the return address: the variable we
want to flip is simply **next to** the buffer in memory.

## Exploitation

No canary stands between the buffer and the VIP flag, so we don't need any offset
precision or return-address control: typing **more than 32 characters** spills
into the VIP flag and sets it non-zero.

Running the binary locally with a long line of `1`s confirms the path — VIP mode
triggers and it tries to read the flag (absent locally):

![local run, flag.txt not found](vn-local-run.png)
_The overflow flips VIP on; locally there's no `/flag.txt`, so it just reports it missing._

Against the remote instance, the same input flips VIP and `win()` prints the flag:

```bash
nc challs.unitedctf.ca 57524
# then enter a long run of 1s (> 32 chars)
```

![remote run, VIP mode and the flag](vn-remote-flag.png)
_"MODE VIP : article offert !" — the flag is dispensed._

## Takeaway

An unbounded `read`/`gets` into a fixed stack buffer lets input overwrite whatever
sits next to it. Here the neighbour is a VIP flag, so the overflow is a one-liner —
no shellcode, no ROP. Bound the read to the buffer size and the bug disappears.
