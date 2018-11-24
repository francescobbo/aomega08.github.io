## The first CPU instructions

How does a CPU work? More or less like this:

1. Read an instruction from memory at the address pointed by the PC register (Fetch)
2. Understand what the instruction is (Decode)
3. Run the instruction (Execute)
4. Increment the Program Counter
4. Repeat

This is clearly not the full story, but let's assume it is for now.

At boot time the PC register is set to BFC0_0000, or the beginning of the BIOS
area, in the uncached segment KSEG1.

We need something to execute there, and luckily we already prepared the memory
controller with the code to load a PS ROM file. You can surely easily extract
this firmware from your own PS, or ask your friend to do so.

I've put mine, usually called SCPH1001.BIN, in the working directory of the
emulator.

### MIPS code!

Using a disassembler, or by hard work on the R3000 instruction set, let's see
the first few instructions:

~~~
3c080013	lui	r8, 0x13
3508243f	ori	r8, r8, 0x243f
3c011f80	lui	r1, 0x1f80
ac281010	sw	r8, 0x1010(r1)
00000000	sll	r0, r0, 0
24080b88	addiu	r8, r0, 0xb88
3c011f80	lui	r1, 0x1f80
ac281060	sw	r8, 0x1060(r1)
00000000	sll	r0, r0, 0
00000000	sll	r0, r0, 0
...
00000000	sll	r0, r0, 0
00000000	sll	r0, r0, 0
0bf00054	j	0xbfc00150
00000000	sll	r0, r0, 0
~~~

So here we have `lui`, `ori`, `sw`, `sll`, `addiu` and `j`. Let's see what do
they do. But first let's talk registers.

#### MIPS Registers

A MIPS Processor has 32 registers. One of them is special, `r0`. It always reads
as 0, and writes to it are ignored.

All of them have a second name, which makes it easier to understand their role,
like a0, v1, t3, sp, ra. But for the purpose of emulation we can refer to them
using their number, as we don't really care how they are used. So r0 to r31 it
is.

#### LUI: Load Upper Immediate

This instruction sets the high 16 bits of a register with the immediate value
provided, and the lower bits to zero.

This is necessary because we can't load an immediate 32-bit value into a
register in a single operation, as the instruction size itself is 32-bit. By
using all of them for the value, there would be no bits left to encode the
"LOAD" instruction itself.

So roughly `lui r8, 0x13` translates to `r8 = 0x130000`.

#### ORI: OR Immediate

With this instruction we do a OR operation of the register with an immediate
value, and store this in a, possibly, different register.

A LUI followed by a ORI (or a XORI or an ADDI), is a common pattern to load
a 32-bit value in two operations.

`ori r8, r8, 0x243f` means `r8 |= 0x243f`. These first two instructions
effectively set the `r8` register to 0013_243F.

#### SW: Store word

This operation will write the value of a register to a specific address. This
address is usually a memory location, but can also be a configuration register
(like in this specific first case) or an I/O port.

`sw r8, 0x1010(r1)` will write the contents of register `r8` at the memory
location pointed by `r1 + 0x1010`.

Since in the previous instruction we set `r1` to 1F80_0000, the BIOS is writing
the value 0013_243F at the address 1F80_1010.

This address is quite well outside the 2 MB RAM and it is indeed a memory
configuration register itself. Let's ignore this for now.

#### SLL: Shift Left Logical

Nothing too fancy here, just shift the value of a register by the number of
bits specified, and write the result in another register.

`sll r0, r0, 0` means... `r0 = r0 << 0`. Well interesting. This is a No
OPeration, or NOP. `r0` is also immutably zero, so the CPU is effectively just
wasting some time...

Later on there will be 20 more NOPs one after the other, I guess because the CPU
is stalling, giving time to the RAM controller to reconfigure itself after
writing to it's important registers.

#### ADDIU: Add Immediate Unsigned

An addition of a register with an immediate, stored in another register.

In this case `addiu r8, r0, 0xb88`, means `r8 = r0 + 0xb88`. But wait, `r0` is
constant zero, so this is actually a simple assignment of 0xb88 to `r8`.

#### J: Jump

An old-style unconditional jump to a target. (A lot) More on this later.

### Instruction encoding

There are different ways an instruction can be encoded. But it all start always
with the 6 most significant bits. That is, by reading those bits you can
understand what operation you have to execute... most of the time. A couple
of values will need further processing.

So we could use a gigantic switch/case with all possible values of those 6 bits
and perform the appropriate operation, like this:

~~~
switch (instruction >> 26) {
  case 0:
    // do something
  case 1:
    // do something else
  // Many more cases
}
~~~

To be fair I'm not a big fan of this solution as it would make the code quite
hard to read. Luckily there's an alternative.

### Implementing the MIPS

First of all I'll have a 32-bit value to represent the Program Counter and an
array of 32 32-bit values which will be the CPU registers. This array will be
called GPR (general purpose registers).

For every instruction I will take the top 6 bits, and use them as an index to
an array of functions that will actually perform the operation.

It looks more or less like this (drafted pseudocode, not valid):

~~~
void doSomething() {
    //...
}

void doSomethingElse() {
    //...
}

// Many more of these

void (*operations)() = { doSomething, doSomethingElse, ... }

void cpuRun() {
    u32 instruction = memoryRead32(pc)
    u32 op = instruction >> 26

    operations[op]()

    pc += 4
}
~~~

The actual implementation of these first few instructions will be like this:

~~~
void lui() {
    gpr[target] = immediate << 16;
}

void ori() {
    gpr[target] = gpr[source] | immediate;
}

void addiu() {
    gpr[target] = gpr[source] + immediate;
}

void sw() {
    u32 address = gpr[base] + offset;
    memoryWrite32(address, gpr[source]);
}

void j() {
    u32 destination = pc + immediate;
    pc = destination;
}
~~~

Now, there are more quirks that we have to be careful of, but for now this is
enough.

Instead of using bit shifting and masking magic, I will use a struct with a
bitfield to access the different parts of an instruction. This may make the
code easier to read.

### Accesses to 1f80xxxx

So, we got our first "memory" writes to 1F80_1010 and 1F80_1060. Again, this are
not a real memory locations, so we need to handle this separately. For now, I'll
just ignore the writes, and cross my fingers, but putting a large TODO item.

### "SPECIAL" instructions

`SLL` is the first instruction we meet that uses operation 0. This single opcode
actually encodes 64 more instructions. To choose which subinstruction we're
handling we need to look into the 6 least significant bits.

For some reason this category of instructions is called SPECIAL.

### Another weird write

The next memory write will be to location FFFE_0130. This is the only memory
access that would happen in KSEG2 (if it is a thing on the PS). This special
register contains configuration for the CPU caches. Again, for now let's ignore
this.

### COP0 instructions

The next instruction that will fail is a COP0 related one. Let's stop here for
now.

## Show me the code!

You can checkout the code at this stage on [Github](https://github.com/aomega08/psemu/tree/73d515a73a0fb22c52d902e47c757e17cdb17a89).
