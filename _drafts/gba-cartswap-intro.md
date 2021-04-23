---
layout: post
title: Gameboy Advance Cartridge-Swapping
tags: gba embedded
---

This is the first in a planned series of write-ups on a recent project I did.
This post will serve as an overview of the goals and high-level technical background.

You can find the code (and its history) at [my GitHub repository](https://github.com/jeresch/gba-savedumper).

## Setting the Stage

![Barley and my modded GBA](/assets/bar-and-gba.JPG){:.inline-image}

I was born in 1995, just on the generational border between the Millenials and Gen Zers.
My introduction to video games was probably through television ads, but my first memories with them is with Pokemon Crystal, played on a Gameboy Advance (GBA) with its backwards-compatibility with the previous Gameboy (GB) and Gameboy Color (GBC) games. [^1]

Fast-forward 20ish years to today, and like many nostalgic video game enthusiasts I recently took out my old collection of GBA games.
Several of these cartridges had save files intact, representing many hours spent as a child engrossed in other worlds.
To preserve these memories, it would be necessary to back up this save data on an external medium.

## Existing Tools

This is not the first time someone has felt nostalgia for old video games, nor is it the first time a GBA save backup tool has been attempted.
Existing tools work really well, but I wanted to create something of my own, and learn along the way.
In the scenario the reader wishes to back up GBA saves though, I highly recommend one of two techniques I know of.

The first (and most convenient) technique requires access to a Nintendo DS or DS Lite, as well as a reprogrammable DS flash cartridge.
In short, [a homebrew program][GBA Backup Tool Download] is loaded and run on the DS with the flash cart, and this homebrew application reads and dumps save data from a GBA cartridge inserted into the DS slot 2.
An [excellent tutorial][GBA Backup Tool Guide] is available, and I'd recommend reading it.
This method is possible because Nintendo designers allowed a DS to read from both slot 1 (DS) and slot 2 (GBA) at once, which was utilized in some [accessories][DS Slot 2 Accessories] and [GBA games][DS-enabled GBA Games].

The second method is to use a specialized cartridge dumper/flasher such as [GBXCart].
I haven't used this method, but it seems to work well, albeit at some cost for single-purpose hardware.

## The Approach

The GBA doesn't have the same memory protections we expect on modern operating systems, and the CPU can execute code residing almost anywhere in the memory address space.
Commonly, much of the code of a game resides in the cartridge-hosted ROM, but there is access latency associated with executing code from the cartridge.
To improve performance, individual subroutines are sometimes relocated (manually) to the IWRAM (in-CPU) or EWRAM (on-board) memory, and executed from there.

When a game cartridge is yanked while a game/program is running, subsequent reads from ROM return junk data, and so code located in ROM is no longer executable by the CPU.
However, code residing in one of the two RAM blocks can continue executing, and if another cartridge is inserted, the CPU can once again interact with it.

This technique is known as "cartridge-swapping," and can be a little tricky to work with, but in principle allows ACE, the holy grail of exploits and hacking.
Of course, we aren't hacking in a criminal sense here, but the possibilities are exciting.
See writeups by others [here][Cartswap writeup] and [here][Cartswap writeup 2].

To accomplish save-dumping, it seems the following sequence should be possible:

1. load a small program into RAM from a custom ROM
2. swap to an OEM cartridge
3. rip the save data into RAM
4. swap back to the custom cartridge
5. dump the in-RAM save data to the custom cartridge
6. back up the custom cartridge (easy if using an sd card flash cartridge)

## Next time

In the next post, I will briefly discuss the GBA development tool landscape, and explore what is necessary to allow code relocation from ROM to RAM at runtime.

[^1]: Interestingly, backwards-compatibility is achieved almost entirely with real hardware rather than software emulation (common in more modern systems). As a result, playing older GB and GBC games usually uses _less_ power than the native GBA games.  GB and GBC emulators that run as native GBA applications [do exist][Goomba Color], but have some drawbacks (like higher power draw).

{% include gba-abbreviations.md %}

[GBA Backup Tool Download]: https://digiex.net/threads/gba-backup-tool-backup-gba-saves-dump-a-gameboy-advanced-rom-using-a-nintendo-ds.9921/
[GBA Backup Tool Guide]: https://projectpokemon.org/home/forums/topic/41730-managing-gba-saves-using-gba-backup-tool/
[DS Slot 2 Accessories]: https://en.wikipedia.org/wiki/List_of_Nintendo_DS_accessories
[DS-enabled GBA Games]: https://nintendo.fandom.com/wiki/List_of_Nintendo_DS_games_with_GBA_connectivity
[GBXCart]: https://www.gbxcart.com
[Goomba Color]: https://www.dwedit.org/gba/goombacolor.php
[Cartswap writeup]: https://eldred.fr/pokexploits/cartswap
[Cartswap writeup 2]: https://gist.github.com/ISSOtm/3008fd73ec66cb56f1caecfcc8b6fb6f
