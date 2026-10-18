---
title: "UnitedCTF - Intercepted Transmission 1"
date: 2026-10-11 06:00:00 -0400
categories: [Writeups, Network]
tags: [unitedctf-2026, network, pcap, wireshark, tftp, forensics]
media_subpath: /assets/img/posts/intercepted-transmission/
toc: true
---

## Context

A resistance spy captured a few seconds of traffic from a SelectStar network
before being detected. We're given the capture, `intercepted-transmission.pcapng`,
with one pointed hint: *a certain vendor code looks suspicious*.

## Solution

Opening the capture in Wireshark and scanning the protocols, the interesting
traffic is **TFTP** — the simple file-transfer protocol used in PXE network boot.
Filtering on `tftp`:

![TFTP read requests in Wireshark](it1-tftp-filter.png)
_TFTP read requests for `pxelinux.cfg/default` and `bzImage` — a PXE boot sequence._

The first file pulled is `pxelinux.cfg/default`, the PXE boot configuration.
Right-clicking a packet from that transfer → **Follow → UDP/TFTP Stream**
reassembles the file, and the hint pays off — the flag is sitting in a `# vendor:`
comment:

![followed stream showing the vendor comment flag](it1-stream-flag.png)
_The reassembled `pxelinux.cfg/default`; the flag is in the `# vendor:` line._

## Takeaway

TFTP has no encryption and no authentication, so anything it carries — here a PXE
boot config — is fully recoverable from a capture. Following the stream of the
suspicious transfer reassembles the file and exposes the planted vendor comment.
