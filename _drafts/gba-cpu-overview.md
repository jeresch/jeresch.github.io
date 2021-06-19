---
layout: post
title: An Overview of the GBA CPU
tags: gba embedded hardware
---

This is the second in a series of posts about my work implementing a GBA save dumper with cart-swapping.
Today's post won't get into the specifics of the project, but will instead serve as an overview of the GBA, its CPU, and some related topics that are relevant to the project later on.
This is _not_ intended to be reference material, or even a comprehensive introduction.
Instead, I hope to lay the groundwork for future Googling if any topics strike the reader as worth exploring.

I am an expert in precisely __none__ of the topics below, and would appreciate any corrections. [^1]
That said, I think this post will set the stage for later posts in this series.

## The GBA

All things considered, the GBA is an impressive piece of hardware, considering it was released in 2001 at least.
It runs on two AA batteries, and manages to get close to the graphical fidelity of the SNES.
It has on-board hardware providing backwards compatibility with the original GB and GBC lines, and it's in a relatively compact form-factor.

The biggest modern complaints with the hardware usually center around the sound capabilities or the unlit LCD screen.
The sound system is a significant step back from the SNES, and the circuitry of the GBA introduces significant noise to the audio output, though that can be addressed with [modern hardware mods][GBA Dehum Kit].
In my opinion, the more problematic issue is the unlit (and unreadable) stock LCD screen, which eventually saw incremental improvements in the GBA SP AGS-001 and AGS-101 models.
Today, there are LCD mods available, but unfortunately they tend to significantly increase power draw and decrease battery life.
Mahko[^2] has a [comparison page][Makho LCD Kit Comparison] for different kits available to improve the stock screen.

## The ARM7TDMI CPU Core

The primary CPU in the GBA advance is an [ARM7TDMI core][ARM7TDMI Wikipedia].
Confusingly, this processor runs the ARMv4T ISA (ARMv9 is the most recently announced version).
The most relevant feature (to us) of this architecture is indicated by the "T," and refers to the "THUMB" operating mode and sub-ISA.

### ARM: RISC-y Business

Please skip this section if you have familiarity with RISC vs. CISC ISA approaches at a high level.

ARM is one of (if not the most) commonly used RISC ISAs, and was recently adopted by Apple as the foundation of its M1 chips that are being incorporated into its desktop line.
ARM also dominates the mobile phone space, in part because of its power efficiency.

RISC, in principle, tends to imply a relatively narrow set of instructions that a particular processor architecture can interpret and execute.
For example, a RISC architecture may have instructions similar to the following.

- add $a, $b
- negate $a
- jump to instruction $n
- jump to instruction $n if $condition

Any real architecture would have more instructions more rigorously specified, but these are the sorts of operations one can usually expect to be present.
An [ARM+THUMB ISA cheat sheet][ARM+THUMB cheat sheet] is available, and ARM has many more documents specifying the available instructions in greater detail.

An important practical consequence of the narrow set of instructions is that they can often (usually) be specified unambiguously in a fixed-length encoding.
All ARMv4 instructions fit into 32 bits, which include the opcode, any condition codes, immediate operands, register identifiers, etc. (details not important here)

The relatively simple, fixed-length encoding not only makes things easier on disassemblers, but allows the hardware implementation of the processor to be minimalistic and focused.
There is no need for complicated recompilation into proprietary microcode (like one would expect in an x86 processor), and one can imagine a simple processor just pushing electrons through the circuitry on every cycle.
The consequence of this is that there are workloads where a RISC or ARM processor will underperform a CISC processor, but for many workloads there will be significantly less power use without degradation of performance.

One final note that's not relevant in the single-core processor at hand is the fact that ARM has a relatively weaker memory model than x86, so there are a few more gotchas in multi-thread code that are neither well captured in most programming languages, nor well understood by most programmers.

### Competitive ARM and THUMB Wrestling

Now, let's steep in the "T" mentioned earlier.
ARMv4 is a fixed-length 32-bit ISA, but when suffixed with a "T," it comes with a fun bonus: the THUMB operating mode.
In essence, THUMB is a fixed-length 16-bit instruction set that is even narrower and more constrained than its big brother past the wrist.
Because each THUMB instruction fits into half the number of bits, we have fewer bits to specify opcodes, immediate operands, register identifiers, etc.
These restrictions can be a real pain when hand-coding THUMB, and the [cheat sheet][ARM+THUMB ISA Cheat Sheet] highlights the differences and restrictions.
In my limited experience hand-coding THUMB, the following restrictions were most painful:

- Frequently, only loregisters (general purpose registers r0-r7) can be addressed.
- A restricted range is allowed for immediate operands (often only 8 bits, so 0x00-0xff).
- Predicates/conditions usually can't be specified for non-branch instructions, leading to more branches to small blocks.

There positive tradeoffs for targeting THUMB though, including

- Smaller code size.  Ideally the code would take half as much space, but in reality one ARM instruction needs multiple THUMB instructions to accomplish the same result.
- Lower power usage in some implementations
- Lower memory bandwidth, particularly relevant to the GBA (more next time...).

The details of how to switch the processor between THUMB and ARM modes is not overly important, but it is worth noting that this is not automatic.
Calling a THUMB subroutine when the processor is in ARM mode will certainly lead to heartache, and vice-versa.
When not using a sane toolchain, or when hand-coding assembly, keeping track of when it is necessary to switch between modes is a pain.

### Processor Permission Modes

There is another axis of processor modes in addition to ARM/THUMB, but it is less relevant to GBA programming, thankfully.

A core part of the security model of modern general-purpose processors is the different operating modes with access to different instructions.
In x86-derived architectures, these are often referred to as "protection rings."[^3]
The general principle is that there are different permissions levels that specify the capabilities of software running in those levels, and though it is easy to transition from a higher level of permissions to a lower one (for example when switching to a user application from an OS kernel routine), transitioning back to a higher level of permissions is done under very controlled conditions (such as a kernel-managed interrupt handler or system call implementation).

ARM processors have a similar model of security, with [several modes][ARM operating modes] the processor can run in, and each can have its own execution state (such as separate stack and register contents).
However, since the GBA is a single-user embedded environment without high-risk network connectivity, most code will run in the privileged System mode.

## The Memory Address Space

The [GBA memory map][GBA Memory Map] is well documented, and I won't cover it in any depth.
That said, if, like me, most of your programming has been in OS-hosted environments on general-purpose processors with MMUs and virtual memory, not to mention standard library `malloc` and `new`, it might come as a surprise that (for the most part) the mapped memory regions are accessible without too much fuss, albeit with quirks, like VRAM only allowing 16-bit accesses.
The drawback to this flexibility is that the user has to explicitly set up their own stack, and choose where global variables live.
There is no `malloc` implementation (or standard library) available, so this is a very different beast to user-space programming on Linux for example.

## Other Arcane Processor Rituals

There are several odd dances that are necessary to accomplish common tasks on the GBA.
These will be dealt with in more detail as necessary in future posts.

### IO Registers

In addition to the ARM general-purpose registers, the GBA has several "I/O registers," which are pretty conveniently mapped into the memory space starting at `0x040000000`.
These can mostly be accessed like any other memory location from C or assembly, though the `volatile` keyword is required most of the time for correctness in C.
These registers have many bit flags packed inside, which sometimes have unusual behavior.
For example, the 16-bit `KEYINPUT` register at `0x04000130` has one bit for each button on the console, and the bit is `0` when the button is activated, and `1` when the button is not activated.
Or, a little more perplexing from a C programmer perspective is the `IF` register at `0x04000202`.
This register has a bit per possible interrupt type, and upon an interrupt the corresponding bit is set to `1`.
In order to "acknowledge" and clear that flag, surprisingly a `1` has to be written to that bit, rather than the `0` I originally expected.

### VRAM and Video modes

As mentioned before, VRAM must be written to in 16-bit segments.
The meaning of those 16 bits is not constant though, and depends on the active video mode (numbered 0-5).
Each video mode has a couple different properties from the other, choosing a combination of

- Bitmapped or sprites and tiled display
- Width and height of display area
- "Inline" 15-bit RGB color or 8-bit indexes into a palette of color values
- Single VRAM page or 2 flippable pages
- Affine sprite transformation abilities

It wasn't until this project that I really understood the point of using sprites in retro video games.
Like today's GPUs are highly optimized to display many polygons on screen with shaders applied, yesteryear's graphics units were optimized for sprites and affine transformations.

I _highly_ recommend [Retro Game Mechanics Explained] for some really interesting explanations of such details.

### Hardware and Software Interrupts

### Cartridge Save Data

https://densinh.github.io/DenSinH/emulation/2021/02/01/gba-eeprom.html
https://dillonbeliveau.com/2020/06/05/GBA-FLASH.html
https://zork.net/~st/jottings/GBA_saves.html
https://www.gbadev.org/docs.php
https://mgba-emu.github.io/gbatek/#gbakeypadinput
https://www.coranac.com/tonc/text/objbg.htm

## The BIOS

[^1]: Please see my [about page](/about) for a brief overview of my credentials.  Aside from those, I can only stand on the authority of a hobbyist who spends too much time on this stuff.
[^2]: Makho is a huge contributor to the GBA fandom.
[^3]: See [this amazing talk][Memory Sinkhole Talk] for a mind-blowing circumvention of the protection ring security model.

[GBA Dehum Kit]: https://handheldlegend.com/collections/game-boy-advance-gba/products/game-boy-advance-wire-free-dehum-dehiss-kit?variant=39312239952006
[Makho LCD Kit Comparison]: https://gameboy.github.io/wiki/backlightmods
[ARM7TDMI Wikipedia]: https://en.wikipedia.org/wiki/ARM7#ARM7TDMI
[ARM+THUMB Cheat Sheet]: https://developer.arm.com/documentation/qrc0001/m/
[Memory Sinkhole Talk]: https://www.youtube.com/watch?v=lR0nh-TdpVg
[ARM operating modes]: https://developer.arm.com/documentation/ddi0344/k/programmers-model/operating-modes
[GBA Memory Map]: https://mgba-emu.github.io/gbatek/#gbamemorymap
[Retro Game Mechanics Explained]: https://www.youtube.com/channel/UCwRqWnW5ZkVaP_lZF7caZ-g

{% include gba-abbreviations.md %}
