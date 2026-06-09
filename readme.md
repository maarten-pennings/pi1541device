# Pi1541

My take on the Pi1541, the cycle exact Commodore 1541 disk drive emulator that runs on a Raspberry Pi.


## Setup SD card

### Step 1

Format SD card to FAT32

### Steps 2, 3, 4

It seems that on a Raspberry Pi (up to version 3), the GPU takes care of the boot process.

- The Bootloader `bootcode.bin` (50 kbyte). 
  This is the first code the Raspberry Pi’s GPU reads from the SD card.
  It initializes the memory and prepares the system to load the GPU firmware.

- The GPU firmware `start.elf` (3 Mbyte) is the "operating system" for the GPU. 
  It initializes the hardware clocks, voltages, and the display output. 
  It is responsible for loading the kernel `kernel.img` into memory and telling the CPU to start executing it.

- The memory map `fixup.dat` (7 kbyte) is a companion to `start.elf`.
  It contains configuration data partitioning the RAM between the GPU and the CPU.


### Step 5

- The firmware `kernel.img` (500 kbyte).
  In the Raspberry Pi, the filename `kernel.img` is the default name the GPU (`start.elf`) 
  looks for to start the main processor. It usually points to the Linux kernel but it can 
  be any compiled binary. The Pi1541 is a bare metal firmware. This means that the code runs
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

- User options `options.txt` (6 kbyte) for the emulator `kernel.img`.


### Step 6, 7, 8

Commodore binaries.

- `dos1541.bin` firmware of the original Commodore 1541 disk drive.
- `dos1581.bin`  firmware of the original Commodore 1581 disk drive. I skipped this one.
- `chargen.bin` optional; font used by emulator on HDMI display.


### Step 9 

- `prg` and `d64` files go in (subdirectories) of `1541` directory 
  (which is in the root of the SD card).


## Links

- The official [documentation](https://cbm-pi1541.firebaseapp.com/) by Steve White.
- The GitHub [repo](https://github.com/pi1541/Pi1541) from Steve White.
- Older versions of [pi1541](https://cbm-pi1541.firebaseapp.com/whatsnew.html#oldversions.:~:text=the%20wrong%20folder.-,Old%20Versions,-.).
- A Pi1541 [manual](https://www.combitronics.nl/download/pi1541%20Manual.pdf) by Michael Long.
- For commands (like `cd`) see the GitHub
  [repo](https://github.com/pi1541/Pi1541/blob/master/docs/IEC%20Commands.md) or
  the SD2IEC [user manual](https://c64os.com/post/sd2iecdocumentation) on C64OS.

(end)
