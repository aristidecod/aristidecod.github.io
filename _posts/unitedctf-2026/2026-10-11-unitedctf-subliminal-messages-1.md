---
title: "UnitedCTF - Subliminal Messages 1"
date: 2026-10-11 05:00:00 -0400
categories: [Writeups, Steg]
tags: [unitedctf-2026, steg, terraria, tedit]
media_subpath: /assets/img/posts/subliminal-messages/
toc: true
---

## Context

SelectStar hides subliminal messages inside the media it produces. The file handed
out is a `.wld` — a **Terraria** world save. Opening it with **TEdit** (the
Terraria map editor) renders the whole map:

![the Terraria world opened in TEdit](sm1-map-overview.png)
_The full world in TEdit — terrain, ores, biomes, nothing obvious at a glance._

## Solution

Panning across the surface, letters are scribbled here and there, laid out left to
right. Hovering one shows what it's made of: the letters are drawn in a single tile
type — **Gold Ore (tile 8)**:

![a letter F drawn in Gold Ore](sm1-letter-goldore.png)
_An `F` built out of Gold Ore tiles; the status bar confirms `Gold Ore (8)`._

They blend into the terrain because the world is full of other blocks. TEdit has a
**tile filter** — enabling it for **Gold Ore** only and hiding everything else
makes the scattered letters pop against an empty background:

![TEdit tile filter isolating the Gold Ore letters](sm1-filtered-letters.png)
_With only Gold Ore shown, the letters (`F L A G …`) stand out, already in order left to right._

Reading them left to right gives the flag.

## Takeaway

A game save is just a structured file, and anything can be encoded in its map
data. The letters were hidden in plain sight by being the same tile as the
surrounding ore — isolating that one tile type with TEdit's filter is all it takes
to read the message.
