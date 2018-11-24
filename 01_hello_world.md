## Chapter 1: Hello world

The PS1 is a quite simple computer.

### The CPU

It has a custom made **MIPS R3000A** CPU, clocking at exactly 33,868,800 Hz (or
almost 34 MHz). This CPU has been stripped of a Floating Point Unit. All the
math here is done using integers only.

On the other hand an extension unit has been added called the *Geometry
Transformation Engine* (or **GTE**). This coprocessor making 3D calculations
orders of magnitude faster than if they were run using standard operations. You
can think of it like the SSE or AVX extensions for Intel CPUs.

### The RAM

It comes with a huge **2 MiB RAM**. Considering that a CD-ROM could hold 700 MiB
of data, you can imagine how many times a game would go back and forth to the CD
to load textures, levels and stuff.

### The GPU

It also has a quite primitive GPU that only supports 2D operations, with a
**1 MiB VRAM**. This memory cannot be addressed by the CPU directly and it
is where the final pixels are drawn for the display to pick up.

### Other devices

It naturally also comes with a handful of other devices that I haven't
investigated yet, like:

* The Sound Processing Unit
* The CD-ROM Drive
* Multiple I/O ports for Controllers and Memory Cards
* Probably more!

## How does an emulator work?

Pretty much just like any computer does. You load the executable in memory,
then you start the CPU fetch / decode / execute loop. You just do all of this
in C++ instead of using transistors. Like this:

~~~
uint32_t programCounter = 0;

while (true) {
  uint32_t instruction = memoryRead32(programCounter);

  switch (instruction) {
    case 0:
      // do this
    case 1:
      // do that
    // ...
  }

  programCounter++;
}
~~~

All the memory read and writes happen on a simple RAM-sized C array. Like this:

~~~
char *memory = new char[2 * 1024 * 1024];

uint32_t memoryRead32(uint32_t address) {
  // Cast the pointer to the size
  uint32_t *ptr = (uint32_t *) &memory[address];
  return *ptr;
}
~~~

## Shall we begin?

First of all, I've setup a tiny C++ development environment. I'm not using
CMake, autoconf or all the stuff. If you have the `build-essentials` package it
should be enough.

To build it just run `make`, the executable will be at `out/psemu`.

Nice to see I can still write compilable C++ and Makefiles after all this years.

## Show me the code!

You can checkout the code at this stage [Github](https://github.com/aomega08/psemu/tree/acab2656658fe1f852f587cc8f4931331d314f47).
