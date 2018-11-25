---
title: MIPS Delay Slots
---

The MIPS architecture is weird. There are so many ways it makes a developer's life hard. It exposes underlying hardware details that we should not be aware of. Or care. Like **branch and load delay slots**.

## Superscalar architecture

Let's briefly review how modern (even in the 80ies) CPUs work. You have your usual Fetch / Decode / Operand Assembly / Execute pipeline (usually with more steps in between), and instructions flow from one stage to the next.

But, instead of letting parts of the hardware stall and do nothing while the pipeline runs, it all runs at the same time. As we Execute an instruction, the Operand Assembly stage will be preparing the following one, and the Decode stage the one further beyond, and the Fetch stage will be communicating with memory to fetch the 4th instruction in the line.

More or less.

This is a great optimisation! But it comes with a problem. When a branch is encountered, how can the processor determine what the next instruction will be, if the branch hasn't completed it's execution yet?

Well the answer is: *it can't*. So, the old behaviour was to just fetch the instructions, assuming that the branch would not be taken (eg, keep loading the instructions linearly), and in case you're wrong, discard the entire pipeline and start from scratch. Today, a great amount of engineering is put on **branch predictors** which will make sure the pipeline is loaded with the path *most likely* to be taken. How it works? No one really knows unless you're one of those that built one.

## Branch delay slot

The old MIPS architecture instead decided differently. Their pipeline would always load the instructions following the branch, as if not taken, but... if the branch is taken, the pipeline is **not discarded**. Execution continues!

At least, this only takes place for a single instruction, but it is enough to distorce the space time in incomprehensible ways. For example, imagine the following pseudo-assembly:

~~~
move r1, 10
move r2, 10
beq r1, r2, target
move r2, 11
move r2, 12

target:
    call exit
~~~

Now, after putting the value 10 in the r1 and r2 registers, there's a conditional branch to `target` that is taken if the values of r1 and r2 are the same. They are! So the branch is taken. But the pipeline already loaded the following instruction `move r2, 11`, so that will be executed before performing the jump.

Notice however that you can't simply swap the order of the instructions to understand it's logic. While the instruction is executed *before* the jump takes place, it is still run *after* the condition has been evaluated. That is, moving 11 into r2, doesn't affect the `beq` comparison, r2 is still 10 at that point.

`¯\_(ツ)_/¯`

### Implementing the branch delay slot

We already implemented a `J` instruction without caring for the load delay slot. That was fine, that time, because the load delay slot contained a `NOP` (or rather `SLL r0, r0, 0` - effectively a no operation), but branch delay slots are used a lot, so we need to have this fixed.

A possible implementation:

~~~
void j(u32 target) {
	isJumping = true;
	jumpTarget = target;
}

void cpuLoop() {
	// remember if were NOT jumping before executing the next instruction
	bool wasJumping = isJumping;

	// This may set isJumping to true (J, JR, BNE, BEQ, etc...)
	fetch/decode/execute();

	// If we were actually jumping, the last instruction executed was
	// a branch delay slot, time to do the jump
	if (wasJumping) {
		isJumping = false;
		pc = jumpTarget
	}
}
~~~

## Load delay slot

A second weirdness we'll have to handle, sooner or later (it doens't seem necessary to run the BIOS, but you can be sure that if something can be exploited, it will be) are `load delay slots`. That is, reading a value from memory takes some time right? So why stalling the execution. Let's keep running the following instructions! Who cares if the following instruction tries to use the destination register, they'll just get the old value.

*I'm sorry?*

~~~
move r1, 10       ; Move the value 10 into r1
load r1, *0x1234  ; Load into r1 the word at memory location r1
move r2, r1       ; Copy the value of r1 into r2
move r3, r1       ; And into r3
~~~

The second instruction will load stuff from the RAM. Let's imagine that the address 0x1234 contained the value 256. The third instruction will copy the value of r1 into r2. On any decent (sorry, I meant recent) CPU, you would expect to have `r2 = 256`, and after that, `r3 = 256`.

What actually happens is that when the third instruction runs, the value of r1 hasn't been updated yet. So, r2 will have the value 10. But... r3 will contain the actually expected value 256.

![Mind blown](https://media.giphy.com/media/26ufdipQqU2lhNA4g/giphy.gif)

But it doesn't end there. What if the instruction in the load delay slot happens to change r1 as well? Eg:

~~~
move r1, 10
load r1, *0x1234
add r1, r1, 10
~~~

So, move 10 into r1, load the value from memory (happens to be 256 again...). Then take the value of r1, increment it by 10 and store it back into r1.

As we said the value read from memory gets written in the register after the following instruction. So maybe it will be 256? You know, we write 20, then overwrite it... No!

That wasn't completely exact... The actual write of the register happens _after_ any register is read, but before any other writes. So the value of r1 is 10 when it's read (thus the addition will be 10 + 10), becomes 256 after committing the previous instruction load, then it's overwritten to 20 as a result of the add operation.

You can imagine it being implemented like this:

~~~
void load(int regIndex, u32 address) {
	u32 value = readMemory32(address);
	delayRegister = regIndex;
	delayValue = value;
}

void add(int destIndex, int srcIndex, int immediate) {
	u32 result = gpr[srcIndex] + immediate;

	if (delayRegister > 0) {
		gpr[delayRegister] = delayValue;
		delayRegister = 0;
	}

	gpr[destIndex] = result;
}
~~~

In fact this is probably very similar to what I may be implementing. But for now, let's avoid this and see how far can we get.

## Back to COP0

In the previous chapter we stopped at a COP0 group instruction. In particular it was an `MTC0` instruction or "Move to Coprocessor 0".

The MIPS R3000A (or any MIPS for all I know) has a small "child" CPU that helps it in the tedious task of running software. In general it can mount up to 4 of them, and usually COP1 is a *Floating Point Unit*, which has been stripped from the PS1 - in fact all computations are with integers! (By the way, the PS1 has a COP2, which is the Geometry Transformation Engine, more on this much later...)

COP0 is quite important though, as it manages the Memory accesses, Hardware Breakpoints, and Interrupts. Normally it takes care of the Virtual to Physical address translation, owning the TLB. All that is off on the PS1.

Plus would it make sense to emulate Hardware breakpoints? We'll see, for now it's a no for me. So what's left are interrupts and a couple of bits about memory, that we'll touch in a second.

## MTC0

So what was that instruction? See, COP0 has 16 additional registers, of which we can safely ignore the majority. But register number 12, that's the Status Register and is quite important.

While, again, we can ignore, for now, most bits, one particular bit is interesting as it detaches the memory bus and connects it to the cache.

Let's review this for a second. A CPU cache should be a small piece of memory that is managed entirely by the CPU, storing recently accessed memory values for a faster second read. It's not something that software should really care about, or consciously write to.

But hey, it's the MIPS again...

Turns out that when bit #16 of the Status Register (Register #12 of COP0) is set, all Load and Store operations are not directed to the main memory (or the RAM), but directly on the L1 CPU cache.

Let me show explain this in code:

~~~
move r1, 0x10000    ; Bit 16 set
mtc0 cop0r12, r1    ; Change the SR
load r2, *1234      ; Read from L1 cache
store *1234, r0     ; Write to L1 cache
mtc0 cop0r12, r0    ; Set the SR to 0
load r2, *1234      ; Read from memory
store *1234, r0     ; Write to memory
~~~

This is insane! But also quite easy to emulate, since we're ignoring caches for now. whenever that bit is set, all load and store operations will be ignored.

## Other instructions

Once we have that sorted out, we encounter a few more missing instructions, namely:

* **`ADD`**: Addition with Registers (with exception on overflow)
* **`ADDI`**: Addition with Immediate value (with exception on overflow)
* **`ADDU`**: Addition with Registers value
* **`AND`**: AND with Registers
* **`ANDI`**: AND with Immediate value
* **`BEQ`**: Branch if Equal
* **`BGTZ`**: Branch if Greater Than Zero
* **`BLEZ`**: Branch if Less Than or Equal to Zero
* **`BNE`**: Branch if Not Equal
* **`JAL`**: Jump and Link (a function call!)
* **`JALR`**: Jump to Register and Link (a function call!)
* **`JR`**: Jump to Register
* **`LB`**: Load Byte from memory (and sign extend it to 32 bits)
* **`LBU`**: Load Byte from memory (and zero extend it to 32 bits)
* **`LW`**: Load Word (32-bits) from memory
* **`SB`**: Store Byte to memory
* **`SH`**: Store Halfword (16-bits)
* **`SLTIU`**: Set on Less Than Immediate Unsigned (compare with value and store result)
* **`SLTU`**: Set on Less Than Register Unsigned (compare with value and store result)
* **`SRA`**: Shift Right Arithmetic (shift right with sign extension)
* **`SUBU`**: Subtraction with Registers

Plus we need MTC0 and MFC0 (Move To/From Coprocessor 0 register).

## Other Hardware Ports

While implementing all these, you will notice writes to some more 1F80_xxxx locations. Most of these we can ignore as usual, with a TODO:

* 1000, 1004, 1008, 100c, (1010 was taken of earlier), 1014, 1018, 101c, 1020, can go for now.
* 1d80 to 1d86 are interesting as they are the SPU Volume and Reverb registers, being set to 0. Actually the BIOS writes 0 to them a few times, it really wants to make sure there's no spurious audio at boot!
* 1100 to 112c are dedicated to the PS1 Root Counters. These are very important, but they're being set to 0. The BIOS is asking us to ignore them right now, and so we will.
* 2041 seems to be a diagnostic port used by the BIOS to report the current status during the boot. It can probably be attached to a 7-segments display.
* Port 1070 or `I_STAT` is very important: hardware devices set bits in it when they want to trigger an interrupt. Software writes 0s in those bits to acknowledge them. Software cannot write 1s (or better, 1s are ignored, 0s clear bits).
* Port 1074 or `I_MASK` is the interrupt mask. An interrupt is only triggered when it's bit is set in both `I_STAT` and `I_MASK` (the first saying an event occurred, the second allowing it to fire)

So let's store the values of these two (remember the special writing logic for `I_STAT`), and ignore the others, or maybe attach a 7-segments display to 2041...

## Enough for today

After all this, the CPU stumbles on an SLTI instruction which I haven't implemented. But before doing so it writes to 1F80_10F0 (DMA Master), things are getting interesting and to... 1F80_1DAA (SPU Control), enabling the SPU. Wohaaa.

## Show me the code!

You can checkout the code at this stage on [Github](https://github.com/aomega08/psemu/tree/8dca5b02da2603f175afb6231a28a6dd561bf39c).
