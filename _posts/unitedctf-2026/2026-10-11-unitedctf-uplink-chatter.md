---
title: "UnitedCTF - Uplink Chatter"
date: 2026-10-11 08:00:00 -0400
categories: [Writeups, Misc]
tags: [unitedctf-2026, misc, morse]
media_subpath: /assets/img/posts/uplink-chatter/
toc: true
---

## Context

A transmission intercepted on SelectStar's telecom network, provided in two forms:
a text transcription and an audio recording. The hint — *old habits die hard in
telecom* — points at Morse code.

## Solution

The text transcription is a string of dots and dashes: Morse code. No need for
anything fancy — I uploaded the file to [morsecodetranslator.com](https://morsecodetranslator.com/),
which decodes it straight to the flag:

![Morse decoded to the flag](uc-morse-decode.png)
_The dots-and-dashes transcription decodes directly to `FLAG-…`._

The audio recording is just the same message in a second form — decoding either one
gives the flag.
