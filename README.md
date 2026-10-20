MiSTer to MEGA65 porting: lessons learnt
========================================

Practical notes for porting a MiSTer core to the MEGA65 with the
[MiSTer2MEGA65](https://github.com/sy2002/MiSTer2MEGA65) (M2M) framework V2.0.1. They come from
[MegaST](https://github.com/freemchr/MegaST), the Atari ST/STe core (October 2026), and are written
so that they apply to other cores. File names like `keyboard.vhd` point to MegaST examples.

The reference for the framework itself is the
[Ultimate MiSTer2MEGA65 Porting Guide](https://github.com/sy2002/MiSTer2MEGA65/wiki/The-Ultimate-MiSTer2MEGA65-Porting-Guide).
This file adds what we only found out by doing it.

Contents:

1. [Mapping: MiSTer parts to MEGA65/M2M](#1-mapping-mister-parts-to-mega65m2m)
2. [Memory and timing](#2-memory-and-timing)
3. [M2M framework pitfalls](#3-m2m-framework-pitfalls)
4. [Vivado pitfalls](#4-vivado-pitfalls)
5. [QNICE firmware pitfalls](#5-qnice-firmware-pitfalls)
6. [MEGA65 hardware facts](#6-mega65-hardware-facts)
7. [Verification before hardware](#7-verification-before-hardware)
8. [Debugging on the real hardware](#8-debugging-on-the-real-hardware)
9. [Repository, releases and testers](#9-repository-releases-and-testers)
10. [Checklist for a new core](#10-checklist-for-a-new-core)


1. Mapping: MiSTer parts to MEGA65/M2M
--------------------------------------

On MiSTer, an ARM CPU (HPS) runs `Main_MiSTer`, which does a lot for the core. On the MEGA65 there
is no ARM: the M2M framework has a small soft CPU (QNICE) with its own firmware. Everything the ARM
did must be redone in the M2M firmware or in hardware.

| MiSTer | MEGA65 / M2M | Notes |
|--------|--------------|-------|
| `<core>.sv` top with `hps_io`, `sys/` | Your own machine module (MegaST: `atarist_m65.sv`) + `main.vhd` | Copy the machine part of the MiSTer top, drop `hps_io` and `sys/`. Keep the MiSTer code a plain copy (not a submodule) and mark every change with a comment. |
| OSD menu string (`CONF_STR`), `status` bits | `config.vhd` `OPTM_ITEMS` / `OPTM_GROUPS`, bits of `osm_control` | The menu line index *is* the bit number. Check what each MiSTer status bit means; some are inverted (MegaST: MiSTer status bit 8 means *colour*, so `mono = not bit`). |
| Saved OSD settings | Settings file on the SD card (e.g. `/<core>/<name>cfg`) | It must exist with exactly `OPTM_SIZE` bytes, else saving is off ("Corrupt config file"). Ship it (all `0xFF` = defaults). |
| ROM loading by the ARM (`ioctl_download`) | M2M auto-load (`C_CRTROMS_AUTO`) and manual load (`C_CRTROMS_MAN`) through a QNICE device | Write a small loader device (MegaST: `tos_loader.vhd`). Expect the firmware to stall if the device never acknowledges. |
| Disk images (`sd_lba`, `sd_rd`, `sd_buff_*`) | `vdrives.vhd` + firmware; the image is buffered in HyperRAM | M2M's buffer is 8 bit, many MiSTer cores use the 16 bit "WIDE" interface: you need a bridge (MegaST: `fdc_bridge.vhd`). |
| Big images (hard disks) read on demand by the ARM | Not in M2M V2.0.1 | MegaST added "unbuffered" vdrives (read/write directly on the SD card) and a fast seek, see 3. |
| Services done in ARM software (e.g. ACSI/SCSI commands, `st_tos.cpp`) | Hardware in the core, or QNICE firmware | MegaST: `acsi_ctrl.sv` executes the ACSI commands in hardware. |
| Remembering mounted images | Not in M2M V2.0.1 | MegaST added it to the firmware (`mntmem.asm`, file `/atarist/stmount`). |
| DDR3 (frame buffers, big buffers) | HyperRAM (shared with the framework, lower 4 MB belong to M2M) | Fine for buffers, **not** for CPU RAM (see 2). |
| SDRAM | SDRAM on the R4/R5/R6 boards only; block RAM on the R3/R3A | The R3 has no SDRAM: plan a block RAM variant with the same bus timing. |
| Scaler (`ascal` in `sys/`) | `ascal` in the M2M av_pipeline (HDMI only) | VGA gets the native signal or the M2M scandoubler. |
| `video_freak` / aspect / crop | M2M crop is **C64 specific** (`crop.vhd`) | Do your own "zoom-in": blank the border in the core (MegaST: `PIX_ACTIVE` from the shifter). |
| USB keyboard (PS/2 codes) | MEGA65 matrix keyboard (`key_num` / `key_pressed_n`) | Map positions in a `keyboard.vhd`; feed the core's native keyboard matrix if it has one (the real IKBD firmware then sees real keys). |
| USB joystick / mouse | Two DB9 joystick ports, 1351 mouse via the paddle (pot) lines | No USB. Mice need their own decoders (see 6). |
| RTC from the ARM | `main_rtc_i` (MEGA65 RTC) | Map it to the core's RTC chip (MegaST: `rp5c15_m65.sv`). |
| UART / MIDI / user port pins | PMOD headers (pass the pins through the board tops) | M2M ties the PMODs off; the tops must be changed (and documented). |
| `Main_MiSTer` reset / cold boot logic | M2M "reset" signals and the firmware CSR reset | Read 3.4 before you use any reset signal. |


2. Memory and timing
--------------------

* **Measure the latency budget in simulation first.** Add N extra wait cycles to the memory
  controller in the simulator and boot the system ROM. MegaST: TOS 1.04 survives 10 extra 96 MHz
  cycles and crashes at 11, so the ST bus needs read data within about 175 ns.
* **HyperRAM is too slow for CPU/video RAM** of cycle exact machines: about 90 ns on average, much
  more while `ascal` bursts. Video, DMA and blitters cannot wait. Use SDRAM (R4-R6) or block RAM.
* **Floppy images in HyperRAM are fine**: sector requests are not timing critical.
* **R3/R3A boards**: no SDRAM. A block RAM variant with exactly the same bus cycle as the SDRAM
  controller (MegaST: `bram_m65.v`, 12 states, read data at t = 6) keeps the core identical; the RAM
  size shrinks (MegaST: 512 KB ST RAM in 224 RAMB36). Keep a separate `.xpr` for it.
* **The MiSTer SDRAM controller needs a Xilinx version**: ODDR for the clock, IOB registers, DQM
  pins (MegaST: `sdram_m65.v`). Its pin constraints go into a separate `.xdc` that only the SDRAM
  boards include.
* **Larger RAM settings than the original machine** (MiSTer extensions like 8/14 MB) can break
  software that assumes the original limits. Test them, but recommend the original sizes.


3. M2M framework pitfalls
-------------------------

### 3.1 Menu (OSM)

* **Inserting or removing a menu line shifts every bit below it.** Update all `C_MENU_*` constants,
  group numbers and any hard-coded indexes (MegaST: the board tops use `main_osm_control_m(48)`).
  Use a script that parses `config.vhd` and regenerates the constants and the settings file.
* **A changed `OPTM_SIZE` changes the size of the settings file.** Ship a new one and tell the users,
  or their settings are not saved any more.
* **The OSM has 25 text rows** (640x400 with a 16x16 font), the option window is `OPTM_DY + 2`, so at
  most 23 menu lines are visible. More lines leave a black bar under the menu.
* **Help/welcome screens**: about 23 lines of up to 38 characters. Count them with a script.
* **Menu heap**: a bigger menu needs a bigger `MENU_HEAP_SIZE` in `m2m-rom.asm`; too small gives
  "Heap corruption: Hint: OPTM_HEAP_SIZE" or a crash when the help opens. `HEAP_SIZE` must shrink by
  the same amount. Check the sum after every menu or firmware change.

### 3.2 Bugs in M2M V2.0.1

* **`ROSM_SAVE` endless loop with more than one virtual drive** (`options.asm`: the vdrive id was kept
  in `R8`, which `VD_DRV_READ` overwrites). Symptom: the whole machine freezes when the menu closes
  or on reset, but only if settings saving is active (settings file with the exact size). Fixed
  upstream on `develop` (bcb34f4); apply it to V2.0.1 (keep the id in `R1`).
* **The firmware never lowers `sd_buff_wr` between bytes.** A bridge must strobe on the register
  write access, not on an edge of `sd_buff_wr`.
* **`FAT32$FILE_SEEK` follows the cluster chain from the start of the file.** Seeking in a 4 GB image
  takes minutes. Scan the chain once at mount (contiguous file = plain calculation, else checkpoints).
* **The FAT32 library shares one sector buffer.** `FILE_OPEN`/`DIR_OPEN` take it over without
  flushing; open extra files through `HANDLE_DEV`, and flush pending writes first.
* **`FAT32$FLUSH` with `R8 = 0` does not set `R9`.**

### 3.3 Things that are C64 specific in the framework

* `crop.vhd` (zoom) is made for 320x200: switch it off and crop in the core.
* `debouncer.vhd` (joysticks): its 1 ms stable time swallows mouse quadrature steps. Set
  `stable_time => 0` and debounce joysticks in the core if needed (MegaST: `joy_lockout.vhd`, a
  lock-out that keeps the press instant).
* The paddle drain pulses (~2 kHz on pins 5 and 9) are for the 1351 mouse; they make optical Amiga
  mice oscillate. Only enable them when the 1351 is selected.
* In 576p (4:3 and 5:4) the `ascal` output window is the full 720 pixels; in 720p it is a centred 4:3
  window. If the picture looks off-centre, check the core's own blanking window first.

### 3.4 Resets

* `main_reset_m2m` is **not** only the user's long reset: the board tops OR the firmware's CSR reset
  into it, and the firmware holds the core in CSR reset while it restores images at startup.
  MegaST: "eject all drives on M2M reset" ejected every image right after it was restored.
* `main_reset_core` includes the CSR reset too. Connecting the vdrives reset to it marks all drives
  unmounted on every core reset while the controllers still use them.
* Use the vdrives reset (from `qnice_rst`) for "the firmware has restarted", and decide explicitly
  which state must survive a short reset (disks inserted, like a real reset button) and which must
  survive a long reset (the firmware restarts: remounted from the remembered list).

### 3.5 Changing the framework

* Change `M2M/` only when unavoidable. Give every change a default that keeps the old behaviour,
  mark it with a comment and list it in one file (`doc/m2m/exceptions.md` in MegaST), so that an M2M
  update can re-apply it. Generic fixes are candidates for upstream.
* The board tops (`top_mega65-r*.vhd`) differ per board; keep them in sync and remember that the R3
  top is different.
* `make_rom.sh` can write constants into `globals.asm`, because `qasm` cannot compute expressions.


4. Vivado pitfalls
------------------

* **Synthesis bug (Vivado 2026.1):** the first call of a VHDL procedure that writes a process
  variable implicitly was turned into a constant (a key always pressed). GHDL and Verilator
  simulated it correctly. Pass variables to procedures as explicit `inout` parameters and don't
  write through `alias`es. If simulation and hardware disagree, suspect synthesis.
* **Language boundary:** no integer ports between VHDL and Verilog (replace e.g. `CEGen.vhd` with a
  Verilog version); no SystemVerilog `struct` ports across files (wrap the module with plain ports).
* **MiSTer code that Quartus accepts but Vivado doesn't**: `wire` assigned in `always` (make it `reg`),
  incomplete sensitivity lists (simulation/synthesis mismatch), implicit nets, stray carriage
  returns inside comments, missing `entity` keyword in direct instantiations, `$readmemh` paths (put
  the `.mem` file into the project).
* **Block RAM demoted to LUTRAM** (Synth 8-5835): check the log after every build; it costs timing
  and LUTs.
* **Vivado rewrites `.xpr` files** (format, absolute paths) whenever it opens them. Never commit a
  rewrite; commit only intended file-list changes. Back up the `.xpr` before a batch build if it has
  uncommitted edits. Keep the file lists of all board projects in sync.
* **Timing failures on the M2M HyperRAM clock path** (`hb_ck_ddr_o_reg` -> `oddr_clk_inst`) came from
  placement luck, not from the core. A hard pblock (`IS_SOFT FALSE`, else Vivado ignores it) that
  keeps the register next to its pin fixed it for all boards.
* After every build check: WNS and WHS >= 0, no new critical warnings, no LUTRAM demotion. Thin hold
  slack is the first suspect for random disk errors.
* `runme.log` is **appended** across runs: count warnings only after the last "Start of session".
* `synth_pre.tcl` runs `make_rom.sh` in a shared folder: start parallel board builds at least 3
  minutes apart, or the firmware assembly fails or a ROM ends up corrupt.
* Never `pkill -f` / `pgrep -f` with a pattern that also matches your own command line.
* The free (BASIC) licence refuses `create_debug_core` (no ILA insertion flow); use JTAG probes
  (see 8).
* The `bit2core` bundled with M2M caps at 4 MB; use the one from
  [mega65-tools](https://github.com/MEGA65/mega65-tools): `bit2core mega65r6 <bit> "<name>" <version> <out.cor>`
  (`mega65r3`/`r4`/`r5` for the other boards).


5. QNICE firmware pitfalls
--------------------------

* `SHL` fills with X, but `SHR` fills with C: clear C before `SHR`.
* There is no `@++R` addressing mode.
* `CMP src, dst` sets N when src > dst.
* `memset`: `R8` = destination, `R9` = count, `R10` = value.
* Test firmware changes in the QNICE emulator with a real FAT32 image built by a script (MegaST:
  `CORE/sim/fastseek`, `CORE/sim/mntmem`), including fragmented files and long file names.
* `f32_mnt_sd` accepts only an MBR whose partition 1 has the type `0x0B` or `0x0C` (FAT32). A card
  that is FAT32 inside but has the type `0x07` (left over from an exFAT format) gives "Cannot mount
  the SD card", although Windows reads it fine (MegaZXNext, real ZX Next card, R3 and R6). Tell
  users how to repair the type without data loss (PowerShell
  `Set-Partition -DiskNumber N -PartitionNumber 1 -MbrType 12`, or diskpart `set id=0C override`).
  For a separate card of another machine (like a real ZX Next card) the SD Association's
  [SD Card Formatter](https://www.sdcard.org/downloads/formatter/) is a good recommendation (cards up
  to 32 GB; larger cards become exFAT). Never tell users to reformat the MEGA65's own SD card: the
  MEGA65 prepares it itself, including its hidden system partition, which a reformat erases.
  A "Cannot mount" message that shows the partition type found would save testers a lot of time.


6. MEGA65 hardware facts
------------------------

* **The pot (paddle) inputs on pins 5 and 9 have no pull-up**, and the core cannot switch one on. A
  released button that only pulls the pin up weakly (Atari mouse right button) recovers slowly; an
  open pin reads exactly like a grounded one. A button on pin 9 that only grounds the pin cannot be
  detected. Guard against "stuck pressed" (MegaST: `rmb_guard.vhd` arms only after it saw the pin
  released). Smart adapters (mouSTer) have an option to drive the pot lines (`activepotlines=true`).
* **Mouse quadrature** from Atari/Amiga mice is too fast for some original controllers when it is
  sampled through the framework; decode it in the core clock domain (MegaST: `quadmouse.vhd`).
* **Joystick switches bounce and chatter while held** (worn sticks like the Suncom TAC-2): a short
  release lock-out (15 ms) with an instant press works well.
* **Keyboard**: C64 layout, F1-F14 with Shift on the same key (F2 = Shift+F1). Offer a positional
  mapping and, if useful, an "as printed" mapping. When the core has to press or hide Shift, let the
  symbol key follow a few ms later, or the original keyboard controller reports the wrong
  combination. The MEGA key is free for an extra layer (keypad, Shift+F-keys). Caps Lock latches.
  The Help key opens the M2M menu.
* **HDMI and DVI monitors**: offer a "DVI mode (no sound)" item; some monitors cannot decode the HDMI
  audio/info packets. 576p modes are the 4:3 and 5:4 options.
* **The 3.5" floppy drive** is a standard DD MFM drive with all signals at the FPGA; using it needs a
  controller that works on the raw bit stream (PLL, MFM, CRC), not the sector-level MiSTer model.


7. Verification before hardware
-------------------------------

Every hardware round trip costs time (build, copy to SD, test, describe). Simulate first.

* **Whole-machine Verilator simulation** with a behavioural SDRAM model, booting the real system ROM
  and writing frames as PPM (MegaST: `CORE/sim/build.sh`, `tb.cpp`). Give it switches through
  environment variables (memory size, video mode, key press at frame N, mouse, floppy and hard disk
  images, reset at frame N). Let the testbench emulate the M2M firmware side (vdrive requests).
* Run the simulated RTL from a copy with the upstream `ifdef VERILATOR` paths disabled, so you
  simulate exactly what Vivado synthesizes.
* **GHDL unit tests** for every VHDL helper (loader, keyboard, drive swap, bridges, guards). Add
  *monitors* that check an invariant on every clock (e.g. "the key never appears with the wrong
  Shift state"), not only end states.
* **Check that a new test fails on the old code.** Otherwise it does not test the fix.
* **Firmware**: QNICE emulator tests (see 5).
* Practical: `stdout` of the simulator is buffered when redirected (logs appear at the end); run it
  from the folder with the `.mem` files, or the CPU never starts; a full boot takes minutes, so run
  variants in parallel.
* Keep simulation results honest: a frame from an old build or a reset state can mislead. Note
  which build and which state a measurement comes from.


8. Debugging on the real hardware
---------------------------------

* **JTAG adapter TE0790** on the MEGA65: Vivado's `hw_server` sees it as a native Xilinx cable.
  Under WSL: `usbipd attach --wsl --busid <id>` after every replug, plus a udev rule for the FTDI
  device. Channel B is the QNICE UART (115200 baud): the M2M debug log.
* **Probes without ILA**: a `BSCANE2` primitive (USER1/USER2) that shifts out a snapshot of internal
  signals (MegaST: 64 bits of the QNICE bus; joystick pins, edge counters and pot values). Read it
  with a small Tcl script and map addresses to the firmware listing (`m2m-rom.lis`). This found the
  `ROSM_SAVE` freeze in one session.
* Load test bitstreams over JTAG (volatile) instead of flashing `.cor` files.
* `mega65_ftp` copies files to/from the SD card over the JTAG serial port, but only while the
  MEGA65 runs its native core (boot the factory slot).


9. Repository, releases and testers
-----------------------------------

* **Never commit copyrighted ROMs** (system ROMs, TOS, Kickstart, ...). Keep local copies in an
  ignored folder; ship a free replacement if there is one (MegaST: EmuTOS).
* Keep three documents: a user README (installation, keys, menu, what is tested), a development
  doc (how the port works, memory map, tests, building) and the framework change list.
* An `AGENTS.md`/`CLAUDE.md` with the rules of the project (menu ripple, memory limits, Vivado bugs,
  board list) keeps AI assistants from repeating old mistakes.
* File headers follow the M2M template (based on ..., powered by MiSTer2MEGA65, port by ...).
* **Release zip**: the `.cor` per board, a README, the core folder with the free ROM, its licence and
  the settings files (correct size, all `0xFF`), plus any other file the firmware needs.
* **Boards**: R6 and R4/R5 differ in pinout (separate `.cor`), R3/R3A needs its own memory variant.
  Never call a board "working" before a tester with that board has confirmed it.
* **Testers** are the biggest help: issue templates (bug, "works"), ask for the exact version,
  board, monitor, ROM and image, and for "does the previous version behave the same?". Ask one
  precise test at a time and thank them; close issues only after a confirmation.
* Don't trust game images blindly: bad dumps exist (check them in an emulator such as Hatari with
  the same ROM), and hard disk game collections often need more RAM or a specific ROM version.
* Mark test builds clearly (`-test1`), keep a user-facing `VERSIONS.md`, and put the version into
  the help screen so testers can report it.


10. Checklist for a new core
----------------------------

1. Read the porting guide; build the M2M demo core for your board to check the toolchain.
2. Copy the MiSTer core (record the commit), write the machine module without `hps_io`, list
   every change.
3. Decide where every memory goes (latency budget measured in simulation; HyperRAM only for buffers).
4. Map every `status` bit to a menu line; write the menu with a generator script; check the
   23-line limit, the help screen size and the menu heap.
5. List everything `Main_MiSTer` does for the core (loading, disk images, services, RTC, saving) and
   plan its M2M replacement.
6. Apply the known M2M V2.0.1 fixes (`ROSM_SAVE`) and decide on crop, debouncer, paddle drain.
7. Set up the Verilator machine simulation and GHDL tests before the first hardware build.
8. Build all boards; check timing, warnings and LUTRAM; keep the `.xpr` files clean.
9. Release with a free ROM and correctly sized settings files; collect tester reports per board.
