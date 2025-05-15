---
title: "Basics of gameboy emulation"
author: "Clotilde Levesque"
date: 2021-05-31T22:17:29+02:00
summary: "How can we emulate a Gameboy microprocessor"
Tags: ["emulation", "architecture", "c"]
Categories: []
DisableComments: false
---

## Introduction

An emulator is a computer program that allows a host system to mimic another system known as the guest.  
Emulation is most often software-based but can also be hardware-based.  
It has a wide range of applications:

- Terminal emulators (software that allows the use of multiple terminals on a single monitor).
- Virtual machines (via hypervisors).
- Printers (often emulate HP-LaserJet for compatibility).
- Software/hardware testing (e.g., in-circuit emulators).
- Game and console emulation, notably the Game Boy.

---

## Principles

To emulate a machine correctly, one must understand its components and how they interact. The main components are:

- CPU (processor)  
- RAM (volatile memory)  
- VRAM (video memory)  
- GPU (graphics processor)  
- I/O (input/output)  
- APU (audio processing unit)

The game is stored in a ROM, which is a sequence of opcodes (8-bit operation codes).

A good emulator is modular. Example:

```c
void write_memory(uint address, uint value) {
    Memory[address] = value;
}

void read_memory(uint address) {
    return Memory[address];
}

void do_instruction(uint opcode) {
    switch(opcode) {
        case(): // do things like write_memory();
    }
}

void execute(void) {
    uint program_counter = 0;
    while("Gameboy powered") {
        uint opcode = read_memory(program_counter);
        do_instruction(opcode);
        program_counter++;
    }
}
```

---

## Game Boy System

- 8-bit CPU similar to Z80, clocked at 4.194304 MHz  
- 8KB of RAM  
- 8KB of VRAM  
- GPU integrated with CPU  
- I/O: buttons, infrared port, link port  
- APU: can play 4 sounds simultaneously

---

## The Processor

### Basics

Basic CPU cycle:

```c
void execute(void) {
    uint program_counter = 0;
    while("Gameboy powered") {
        uint opcode = read_memory(program_counter);
        do_instruction(opcode);
        program_counter++;
    }
}
```

### Registers

- 8-bit registers: A, B, C, D, E, H, L, F  
- 16-bit registers (pairs): AF, BC, DE, HL  
- Special registers: SP (stack pointer), PC (program counter)  
- F register (FLAGS):

```
7 6 5 4 3 2 1 0  
Z N H C 0 0 0 0
```

### Instructions

Seven main opcode families:

- MISC & control: `NOP`, `STOP`  
- Calls & jumps: `JP`, `RET`  
- Load/Store 8-bit: `LD C, B`  
- Load/Store 16-bit: `LD SP, HL`  
- 8-bit Arithmetic/Logic: `ADD A, E`  
- 16-bit Arithmetic/Logic: `ADD HL, BC`  
- Shift/Rotate: `SWAP B`

Example for `ADD A, B` (opcode `0x80`):

```c
void do_instruction(uint opcode) {
    switch(opcode) {
        case 0x80:
            register[A] = register[A] + register[B];
            break;
    }
}
```

### Interrupts

Five types of interrupts:

| Name     | Cause                        | Address |
|----------|------------------------------|---------|
| VBLANK   | Screen refresh               | 0x40    |
| LCDC     | LCD mode change              | 0x48    |
| SERIAL   | Serial transfer              | 0x50    |
| TIMER    | Timer                        | 0x58    |
| HiToLo   | Button press                 | 0x60    |

---

## Memory

### General Memory Layout

| Usage                      | Start | End   |
|---------------------------|-------|--------|
| ROM bank #0               | 0000  | 4000   |
| Switchable ROM bank       | 4000  | 8000   |
| Video RAM                 | 8000  | A000   |
| Switchable RAM bank       | A000  | C000   |
| Internal RAM              | C000  | E000   |
| Echo of Internal RAM      | E000  | FE00   |
| Sprite Attribute Memory   | FE00  | FEA0   |
| Empty Space               | FEA0  | FF00   |
| I/O Ports                 | FF00  | FF4C   |
| Empty Space               | FF4C  | FF80   |
| High RAM (HRAM)           | FF80  | FFFF   |
| Interrupt Enable Register | FFFF  | FFFF   |

### Video RAM

Uses 8×8 pixel **tiles**:

- **Background**: 256×256px, wraps around, no transparency  
- **Window**: 256×256px, higher priority, no wrap  
- **Sprites**: 8×8 or 8×16px, transparent color 0, limited to 40 total / 10 per scanline

### Stack

LIFO structure. Stack pointer (SP) is updated:

- `push`: write at SP then decrement  
- `pop`: increment SP then read  

Used to save register values during interrupts.

---

## What About the Code?

### Structure Representation

```c
typedef struct gameboy {
    uint8_t V[9];
    uint16_t SP;
    uint16_t PC;
    uint8_t *internal_memory;
    uint8_t *screen;
    uint8_t *key_flags;
} gameboy;
```

### Initialization

```c
gameboy *init_gameboy(void) {
    gameboy *gameboy = malloc(sizeof(gameboy));
    if (!gameboy) return NULL;

    gameboy->internal_memory = calloc(8192, 1);
    gameboy->screen = calloc(23040, 1);
    gameboy->key_flags = calloc(16, 1);

    for (size_t i = 0; i < 9; i++) gameboy->V[i] = 0;
    gameboy->SP = 0;
    gameboy->PC = 0;

    return gameboy;
}

void free_gameboy(gameboy *gameboy) {
    free(gameboy->internal_memory);
    free(gameboy->screen);
    free(gameboy->key_flags);
    free(gameboy);
}
```

### Emulation

```c
void write_memory(uint address, uint value) {
    Memory[address] = value;
}

void read_memory(uint address) {
    return Memory[address];
}

void do_instruction(uint opcode) {
    switch(opcode) {
        case 0x80:
            register[A] = register[A] + register[B];
            break;
    }
}

void execute(void) {
    uint program_counter = 0;
    while("Gameboy powered") {
        uint opcode = read_memory(program_counter);
        do_instruction(opcode);
        program_counter++;
    }
}
```

---

## Conclusion

Some parts remain to be implemented (e.g., display, I/O detection, and tests with TDD).  
However, the overall emulator architecture is clear and based on the Game Boy's specification.  
A low-level language like C or Assembly is recommended for this kind of emulator.

---

## Sources

- PyBoy  
- GameBoy Instructions Set  
- Game Boy Programming Manual  
- Game Boy Project  
- Game Boy CPU Manual  
- Furrtek GBASM
