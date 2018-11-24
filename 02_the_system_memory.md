## The System Memory

Before we can start running any CPU instructions, we need a place where we can
store and read these instructions. We need to emulate the RAM.

This should be fairly easy, as the PSX doesn't have a proper Memory Management
Unit (or MMU), so it doesn't support what is today known as *pagination*, but
it still has a few quirks.

At least, it only has 2 MiB of RAM, so it will be really easy to emulate the
physical memory by just allocating a large enough array and shove stuff there.

### The PSX Memory Map

Let's start with the quirks. This will not be a full description of the PS1
memory, as I still don't know everything about it, but you may find more in the
later chapters.

#### R3000A Memory segments

First of all, the MIPS R3000 architecture is designed to split the virtual
address space in 4 areas (or segments), called *KUSEG*, *KSEG0*, *KSEG1* and
*KSEG2*.

The **KUSEG** area is 2 GiB large, goes from 0000_0000 to 7FFF_FFFF and is meant
to be the only one accessible by user-mode software (eg, not kernel, but
applications - or games?).

**KSEG0** and **KSEG1** are both 512 MiB large, the first starts at 8000_0000
and the other at A000_0000. They are both mapped to the first 512 MiB of
physical memory directly (accessing A000_0000 reads physical address 0000_0000).
The big difference between these two is that memory access to *KSEG1* are not
using the CPU cache (reads go directly to RAM).

Finally **KSEG2** is a 1 GiB large memory area which, according to the R3000
documentation, starts at 0xC000_0000 and allows you to read and write directly
at the 3rd gigabyte of physical memory. The PS1 clearly does not have that much
memory (maybe only NSA-grade computers did back in 1989), so we can imagine that
KSEG2 does not exist.

#### The firmware ROM (or BIOS)

The MIPS architecture loads the boot firmware ROM at physical address 1FC0_0000.
It is exactly 512 KiB large, and it's (or is it?) read only.

Interestingly, the boot address of the MIPS CPU is BFC0_0000, not 1FC0_0000.
This address lives in KSEG1, which is not cached.

Again by reading the manual, you learn that CPU caches could be in a random,
inconsistent state at boot, so the firmware starts it's execution in a slower,
but safer memory area, and should relocate itself to KSEG0 as soon as caches
are cleaned.

#### More PSX specific

Even though KUSEG has a respectable 2 GiB size, the PS only has 2 MiB of
physical memory, which is available directly from address 0000_0000 onward.

An interesting feature (or bug? And why would you do this?) is that the first
2 MiB of memory are mirrored 3 more times until the 8th MiB. This means that
a read at address 0020_0000 is effectively reading byte 0 of physical memory.
It seems that this behaviour can be disabled.

At 1F80_0000 we have most memory mapped I/O ports (as in writing and reading in
these locations has effects on the I/O devices).

## Emulating the PS1 memory

As announced, I started with two simple arrays, one for the RAM and another for
the 512 KiB ROM.

I'm using C++ templates for memory accesses of the 3 possible sizes (1, 2 or 4
bytes).

Right now, for every memory access I will simply ignore bits 29 to 31, bringing
everything to `KUSEG`. I'm not sure I can even emulate the caches (make things
slower in KSEG1?).

## Undefined behaviours

The PS1, just like any hardware is full of unknown/undefined behaviours that are
nonetheless exploited by games. For this reason I will play conservatively,
adding assetions here and there, making the emulator crash for events I could
probably ignore, just to see which games are using it and how to emulate that
event as closely as possible to the orignal hardware.

For this reason, I'll put a guard for any memory access in the "dark undefined
areas".

## Show me the code!

You can checkout the code at this stage [Github](https://github.com/aomega08/psemu/tree/9757e72828d13fa616b9e0bb804912c9f4f30df8).
