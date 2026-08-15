# SD card


## Setup SD card

### Step 1

Format SD card to FAT32

### Steps 2, 3, 4

It seems that on a Raspberry Pi (up to version 3), the GPU takes care of the boot process.
When the Raspberry Pi powers on, the bootloader burned directly into the hardware (ROM) wakes up. 
It only knows how to look for a file named `bootcode.bin` on an SD card with FAT16 or FAT32.
We get boot files from [raspberrypi's GitHub](https://github.com/raspberrypi/firmware).

- The Bootloader `bootcode.bin` (50 kbyte). 
  This is the first code the Raspberry Pi’s GPU reads from the SD card (into the GPU's cache).
  It initializes the memory and prepares the system to load the GPU firmware.

- The GPU firmware `start.elf` (3 Mbyte) is the "operating system" for the GPU. 
  It reads `config.txt`, sets up the clocks, configures the display/HDMI, splits the RAM between the GPU and the CPU. 
  Finally, it loads your bare-metal binary `kernel.img` into RAM, and releases the ARM CPU from reset so it starts executing code.

- The memory map `fixup.dat` (7 kbyte) is a companion to `start.elf`.
  It contains configuration data partitioning the RAM between the GPU and the CPU, read by `start.elf`.


### Step 5

We get the real firmware from [Pi1541.zip](https://cbm-pi1541.firebaseapp.com/Pi1541.zip).

- The firmware `kernel.img` (500 kbyte).
  In the Raspberry Pi, the filename `kernel.img` is the default name the GPU (`start.elf`) 
  looks for to start the main processor (ARM CPU). It usually points to the Linux kernel but it can 
  be any compiled binary. The _Pi1541_ is a bare metal firmware. This means that the code runs
  directly on the hardware without an underlying operating system (no scheduler, no file system).

  I did not takes latest greatest, which was 1.24, but 1.23, because I want the activity LED
  and that seems [broken](https://github.com/pi1541/Pi1541/issues/206#issuecomment-1162708867)
  in V1.24. Find older versions at the bottom of this [page](https://cbm-pi1541.firebaseapp.com/whatsnew.html#oldversions.:~:text=the%20wrong%20folder.-,Old%20Versions,-.).

- Configuration `config.txt` (41 byte) is read by `start.elf` (the GPU firmware), not `kernel.img`.
  It contains two lines.
  - `kernel_address=0x1f00000`  
    By default, the Raspberry Pi loads the kernel at address 0x8000 but the Pi1541 code is compiled for another address.
  - `force_turbo=1`  
    By default the Raspberry Pi clocks its CPU down when it is idle (dynamic frequency scaling). 
    This locks the CPU at its maximum rated speed constantly.
    The 1541 disk drive is a "real-time" device, it communicates with the C64 using fixed pulse widths.
    If the Pi would clock down, the pulse timings will shift, leading to communication errors.

- User options `options.txt` (6 kbyte) is written specifically for the Pi1541 
  emulator (which buttons, OLED type, how many IEC connectors, etc).

- The directory `1541` is supposed to be the root directory for the "disks" served by the firmware.
  It contains (will later contain) disk images (`.d64`) files, but it may also contain `PRG` files, 
  i.e. C64 (or VIC-20, or ...) binaries that the Pi1541 will also serve as a disk image 
  (with just that file). The initial contents of this directory is a file browser per supported 
  Commodore platform, like `fb64` for the Commodore 64.


### Step 6, 7, 8

Commodore binaries, e.g. from Zimmers [drive roms](https://www.zimmers.net/anonftp/pub/cbm/firmware/drives/new/1541/index.html) 
or [C64 roms](https://zimmers.net/anonftp/pub/cbm/firmware/computers/c64/index.html) or own your 
own PC if you have [VICE](https://vice-emu.sourceforge.io/) installed.

- `dos1541.bin` firmware of the original Commodore 1541 disk drive.
- `dos1581.bin`  firmware of the original Commodore 1581 disk drive. I skipped this one.
- `chargen.bin` optional; font used by emulator on HDMI display.


### Step 9 

- `prg` and `d64` files go in (subdirectories) of `1541` directory 
  (which is in the root of the SD card).

