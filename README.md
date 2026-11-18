# Karateka on the Tandy Color Computer 3: how it was built

*Karateka (1984) by Jordan Mechner is one of the classics of the Apple II. This is how it ended up running on a CoCo 3: first on a 512K machine, then on a stock 128K one. It's the human version of the project notes: the technical story, without the commit-by-commit detail.*

---

## The idea

The Apple II and the CoCo 3 are close cousins from very different families. One has a 1 MHz 6502; the other a 1.79 MHz 6809 with an MMU, a GIME video chip and 16 colours. The goal was simple to state and hard to do: run Karateka on a CoCo 3 at the speed of the Apple, with the same pictures, the same artifact colours, and the same feel.

There are two ways to bring a game across:

1. **Emulate the Apple** on the CoCo: an interpreter, or a translator that runs the Apple program as it is.
2. **Port it:** turn the game's own code into native 6809, and rewrite by hand the parts that talk to Apple hardware.

The first attempt was option 1. An Apple II emulator for the CoCo 3 ("A2CC3") was built: a 6502 interpreter, an ahead-of-time 6502→6809 translator, the Disk II, and a video converter. It worked, and it booted the real disk to the fight, but it ran at about 35% of Apple speed. Not playable.

So the project switched to option 2, in the style of the CoCo 3 Robotron port: a **static transcoding port**. It has no interpreter and no Apple emulation; it is just 6809 code doing what the 6502 did.

> You need your own copy of the Karateka disk. This project contains tools and hand-written code, never the game's bytes, ROMs or listings derived from them. The build reads your `.dsk` at build time.

---

## Step 1: take the disk apart

Everything starts with understanding the disk. For that I used my own Claude skill, the **6502 disassembler skill**:

👉 **https://github.com/reyco2000/6502-dissasembler-skill**

The disk is a crack, in DOS 3.3 sector order, with no catalog and its own boot chain. Tracing the boot code gave the load map:

- **boot0** at `$0801` reads track 0 into `$B600-$BFFF`: the loader and RWTS.
- **The cold load** reads tracks 1-10 into `$0800-$A7FF`, and a **relocator** at `$2100` moves things into place before jumping to the intro at `$B77C`.
- **Each of the 5 stages** is loaded later by a "stage reboot" at `$0300`. It patches the boot PROM code, reboots the disk with a stage number, loads a 64-page block into the hi-res area, and relocates it.

The skill produced annotated listings for:
- the memory image at the hand-over to the game (the "cold image")
- the memory image after each of the five stage loads

Each of these can be reassembled with ca65 back to the exact original bytes. That byte-for-byte check matters: it proves the disassembly describes the real program.

Some useful facts came out of comparing the images:
- **The engine** (`$0800-$9FFF`, about 3,700 instructions) is **identical in every stage**. Only a few self-modified bytes differ.
- **Each stage** brings its own code in `$0200-$07FF` and `$A000-$BFFF`, plus data pages (shapes, tunes) and the hi-res pages.
- **The disassembly is not a complete map of the code.** A lot of executed code is listed as data. So the transcoder had to find code by itself: by recursive descent from the known entry points, plus every instruction a profiling run of the real game executed.

---

## Step 2: the 512K version

### What is transcoded

A Python transcoder (`xcore/`) turns the 6502 code into 6809 assembly, which lwasm assembles:

- **Registers:** A stays A, X lives in B, and Y lives in a memory byte (`RY`). The 6502 stack is the 6809 stack, and the zero page is the 6809 direct page.
- **Flags:** the generator tracks which flags are actually read later (liveness). It handles the 6502's inverted carry after compares lazily, so most instructions become one or two 6809 instructions.
- **Hardware access** (keyboard `$C000`, speaker `$C030`, page flips `$C054/$C055`, buttons, paddles) becomes calls to small runtime handlers.
- **What gets translated:**
  - **The engine** (`$0800-$9FFF`) is translated **once**: about 9.6K of 6809.
  - **The stage code** is translated **once per stage**, five times, since each stage has its own overlay.

Every translated routine is checked against the original. A test harness runs thousands of random cases on a 6502 model and on a 6809 model, and compares registers, flags and memory (`test_engine`).

### What is not transcoded (rewritten by hand)

Some parts of the game are written for the Apple's hardware, and would be slow or meaningless as a translation. These were rewritten in 6809, with the same inputs and outputs:

| Apple routine | CoCo replacement |
|---|---|
| blitters `$1900-$1C78` (draw, mirrored draw, clear) | `nb_*` in `hires.asm`, with pre-shifted tables |
| FillRect, ClipRect, clear-bit-6 (`$0A00-$0AF7`) | `nh_fill`, `nh_clip`, `nh_clrb6` |
| 2-line scroll copy `$0786`, column wipe `$7761` | `nh_scroll`, `nh_colcp` |
| paddle read loop `$6D09` | joystick read through the CoCo DAC |
| VBL wait `$779A` | GIME timer / 60 Hz interrupt |
| tune and sample players `$0D00/$0DC0`, monitor `WAIT` | the same timing on the CoCo's 6-bit DAC |
| boot, RWTS, stage loader, disk reboot | a native loader |

Every native routine must leave **exactly** what the 6502 original leaves, down to the zero-page scratch bytes, because the rest of the (translated) game may read them. `test_native` checks that bit for bit.

### The video trick

The game draws on the Apple hi-res bitmap and also *reads* it: it ORs and ANDs sprites into the scene. So the Apple bitmap stays the "truth", in Apple format. A converter turns the parts that changed into the GIME's 320×192, 16-colour format, artifact colours included. The colours come from two lookup tables indexed by a byte pair and its neighbours' edge bits.

Converting a whole screen every frame would be far too slow. So every native drawing routine records **dirty rectangles**, and the converter only looks there. Inside those rectangles it compares against a "shadow" copy of what is on screen, and converts only byte pairs that really changed.

### The 512K memory map

In 512K, memory was not a concern:
- **6 blocks:** the Apple's 48K of RAM
- **2 blocks:** the engine code
- **one 8K bank per stage:** its code, mapped in when the stage is played
- **2 GIME display buffers + 2 shadows:** a true double buffer, so the screen flips like the Apple's
- **compressed blobs:** the cold image, the five stages and the tables, unpacked at boot
- **a resident runtime block** at `$E000`: interrupts, keyboard, joystick, sound, video conversion

Everything goes to memory: after loading, the game never touches the disk again.

A **fast loader** (`K3.BIN`) drives the WD1793 disk controller directly. It reads sectors in their physical order and draws a title screen with a progress bar, which cut the load from about 75 s to about 21 s.

After a few rounds of profiling and tuning (the blitter, the converter scan, the dirty-rectangle merging), the 512K version played at Apple speed.

---

## Step 3: the 128K version

A stock CoCo 3 has 128K: 16 blocks of 8K. The 512K build used more than 50. The challenge was to keep every speed improvement and make it all fit.

### The new budget

| blocks | what |
|---|---|
| `$30-$33` | **one** GIME display buffer (30,720 bytes); the video converter lives in the last 2K |
| `$34` | the staging log (and the disk track buffer while loading) |
| `$35-$36` | the colour conversion tables |
| `$37-$38` | engine code, plus **the current stage's code** and the blitter's shift tables |
| `$39-$3E` | the Apple's 48K of RAM |
| `$3F` | the runtime |

That is exactly 16 blocks; nothing is spare.

### What was rewritten for 128K

**Stages stream from disk.** There's no room for compressed stage images in RAM. A new disk streamer (`dstream.asm`) reads a track at a time into a buffer and decompresses it straight into its final place. The last frame stays on screen while a stage loads, and game over reloads the cold image the same way.

**One screen buffer instead of two.** This was the hard part. With one buffer, the converter writes to the screen the player is looking at. At first, a scroll step (the scene sliding as the hero walks) took 4-6 frames to convert, and you could see the update sweep across the screen. On 512K the same work happens on a hidden buffer.

The fix:

- **The shadow was dropped.** The converter now compares the new page with the page shown last, which is still in Apple RAM. That freed an 8K block.
- **The free block became a staging log.** Converted bytes are logged there as runs, then copied to the screen in one burst, 7 bytes per `PULU`/`PSHS` pair.
- **The copy is timed to the TV beam.** The changes are sorted top to bottom, and the copy starts either at the vertical sync or at the top of the picture, depending on its size. Each line changes after the beam has drawn it and before it draws it again, so a whole scroll step appears at once.
- **Overlapping rectangles are merged first.** Without a shadow, a pair covered by two rectangles would be converted twice.

**The speed work went further.**
- **Faster converter loop:** it now converts a whole run of changed pairs in one tight loop.
- **Faster intro scroll copy:** it uses stack instructions (`PULU`/`PSHS`) instead of plain loads and stores.
- **Leaner blitter rows:** each row does less setup work, and shapes drawn completely off screen no longer map the screen memory at all.

On the way, the speed was re-measured against the Apple on exactly the same scene, using screenshots to confirm it. An earlier comparison had matched a CoCo fight against an Apple screen with nobody in it.

### What is transcoded in the 128K version

The translated game is **the same** as in the 512K version: the same engine, the same five stage overlays, the same generator. Only the runtime around it changed:
- the memory map
- the disk streamer
- the video converter
- the stage loader
- a few native routines, made faster

That's the nice part of this approach. The game logic is never touched by hand; it is translated from the original and tested against it.

### Where it stands

Measured in MAME against an Apple II model running the same input:

| scene | CoCo 3 128K | Apple II |
|---|---|---|
| a fight, draws per 600 frames | 7,122 | 7,061 |
| attract mode, page flips by frame 9000 | 514 | 514 |
| a scroll step on screen | one burst, timed to the beam | (page flip) |

Tested on XRoar 128K by me: it plays, it scrolls smoothly, and it runs at Apple speed. It has not been tried on real hardware yet; that's next.

---

## Build it yourself

This repository has no game data. You build the CoCo 3 disk from your own copy of the Apple II Karateka disk. You need:
- Python 3
- [LWTOOLS](http://www.lwtools.ca/) (`lwasm`)
- gcc
- MAME with the Apple IIe ROMs `APPLE2E.ROM` and `341-0027-a.p5` (the build runs the real Apple loader on a model Apple II)

The source code and the exact build commands will be published in this repository. In short:
1. Put your disk image in the project root.
2. Rebuild the memory images it loads.
3. Make the stage snapshots.
4. Build `build/karateka_k3_128k.dsk`.
5. In MAME (`coco3 -ram 128k`) or on a CoCo 3: `LOADM"K3":EXEC`.

---

## Tools

- **Disassembly:** [6502 disassembler skill](https://github.com/reyco2000/6502-dissasembler-skill)
- **Assembler:** lwasm (LWTOOLS); ca65 to check the reassembled listings
- **Emulators:** MAME (CoCo 3 and Apple IIe) for automated tests, Lua probes and traces; XRoar for play-testing
- **Python:**
  - the transcoder
  - the build
  - an Apple II model used as the reference
  - the differential tests
- Built with an AI pair-programmer (Claude Code) doing a lot of the tracing, measuring and assembly.

---

## Credits

- **Karateka** © 1984 Jordan Mechner / Broderbund.
- CoCo 3 port by Reinaldo Torres, CoCoByte Club.

---

## Like it? ⭐

If you enjoy seeing Karateka fight its way onto the Color Computer 3, **please star this project on GitHub**, and the [6502 disassembler skill](https://github.com/reyco2000/6502-dissasembler-skill) too. It helps other retro fans find it, and it keeps these ports coming.
