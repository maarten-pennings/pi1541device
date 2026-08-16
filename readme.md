# Pi1541

My take on the _Pi1541_, the cycle exact Commodore 1541 disk drive emulator that runs on a Raspberry Pi.


## Introduction

The _Commodore 1541_ is a 5.25-inch floppy disk drive.
It was _the_ solution for file storage when working with Commodore 64 (C64). 
Over the decades the drives and floppy disks degraded.
Various disk drive replacements where developed. 
One of them is the _Pi1541_, the topic of this repo.

> ![C1541](images/c1541.jpg)
>
> _Commodore 1541 from [Commodore online museum](https://cbmmuseum.kuto.de/floppy_vc1541.html)_
> _and later model (1541-ii) from [Wikipedia](https://en.wikipedia.org/wiki/Commodore_1541)_


An important predecessors of the Pi1541 was the 
[SD2IEC](https://www.c64-wiki.com/wiki/SD2IEC). The SD2IEC 
contains an ATmega644 micro controller from Atmel that 
translates Commodore serial bus (IEC) commands into FAT 
file system operations on the SD card. This allows the C64 
to access files on an SD card.

> ![SD2IEC](images/pcbway-sd2iec.jpg)
>
> _Example SD2IEC project from [PCBWAY](https://www.pcbway.com/project/shareproject/SD2iEC_COMMODORE_64_DISK_DRIVE_EMULATOR_POWERED_from_USER_PORT_MICRO_SD_VERSI_c56a9f83.html)_

The SD2IEC is low-cost and capable to store about 25000 floppies
(of 170 kByte each) on a 4GB SD card. It is small and can be powered 
by the C64 itself (usually via the user port). 
The big drawback is that while the SD2IEC is compatible
with the _file level commands_, it cannot handle the _programming commands_ 
given via the secondary channel 15.

Unfortunately, due to the low-speed of the 1541, many commercial games, demos, 
and developer cartridges come with their own fast loaders. Part of a fast loader 
is software that is uploaded to the 1541 drive (which is a 6502 based 
computer on its own) via the above mentioned programming commands. 
The SD2IEC cannot run that uploaded software.
This results in low software compatibility: a large portion of the C64 
applications (games) will not load from an SD2IEC.

The Pi1541, developed by Steve White, solves this. Instead of an _ATmega644_ micro 
controller, it uses a more powerful _Raspberry Pi_ and a simple add-on (a so-called "HAT"). 
The Pi is powerful enough to provide a cycle-exact hardware emulation of the real 
Commodore 1541 drive. It emulates the drive's internal 6502 CPU, the RAM, 
and the VIA chips down to the exact clock cycle. You even need to download the 
original Commodore ROM image of the 1541 to "program" this virtual 6502.

> ![Pi1541](images/pcbway-pi1541.jpg)
>
> _Example Pi1541 project from [PCBWAY](https://www.pcbway.com/project/shareproject/Pi1541_IO_Adapter__Rev_2.html)_

This solution provides near 100% compatibility because it behaves exactly 
like the original drive. It supports complex copy protections and almost every 
fast loader ever written (e.g., JiffyDOS, Final Cartridge III).

Steve White wrote a "bare metal" application on the Pi. This means his 
emulator does not run under Raspberry Pi OS (Linux) but directly on the 
_Broadcom BCM2837B0_. This results in fast booting, and carefree switching off 
the power. On the down side, the Pi1541 is more expensive than the SD2IEC due to the 
Raspberry Pi board; it is a bit bigger physically; and needs (more) external power.
But most believe these down sides are well compensated by the extra compatibility.


## Concepts

This section explains the key concepts when working with the Pi1541: 
current working directory, three categories of commands, the new `CD` command, 
mounting `.d64` as virtual floppy, the problem with the programming commands, 
and finally _browse mode_ and _emulation mode_.


### Current working directory

The Pi1541 contains a SD card, and it maintains a notion of the 
_current working directory_ on that SD card. Its firmware bridges the 
commands that come from the C64 (over the IEC bus) to the FAT file system 
on the SD card. 

It does understand the _high level command_ such as 
`SAVE "MYGAME"`, `LOAD "MYGAME"`, and even `LOAD "$"` to read the 
(current working) directory. The `SAVE "MYGAME"` creates a file `MYGAME` 
on the SD card, in the current working directory. Similarly `LOAD "MYGAME"` 
reads the file `MYGAME` from the current working directory and sends it back 
to the C64 over the IEC bus. It will be no surprise that `LOAD "$"` reads 
the current working directory and sends that back to the C64.


### Commands 

If you are more than a casual user of the C64 and the 1541 drive, 
you will know that next to those high level commands there are more 
_advanced commands_ that a 1541 will understand. 
The Pi1541 also emulates those. For example `OPEN 1,8,15,"S0:MYGAME":CLOSE 1` 
scratches (deletes) the file `MYGAME` from the current working directory, 
and `OPEN 1,8,15,"R0:YOURGAME=MYGAME":CLOSE 1` renames `MYGAME` to `YOURGAME`.

Recall that the DOS (disk operating system) was not part of the C64 
(would have taken too much memory), rather it was baked into the 1541 itself.
What is part of the C64, is setting up a "data pipe" (open a file) and send 
the (textual) command over that pipe. The 1541 receives the textual command,
parses it and executes it. That is the reason we have advanced commands 
(eg see [c64-wiki](https://www.c64-wiki.com/wiki/Commodore_1541#Disk_Drive_Commands)).


### CD 

You might be wondering how to _change_ the current working directory. 
The developers of the SD2IEC added an advanced command: `CD`. For example 
`OPEN 1,8,15,"CD:SUBDIR":CLOSE 1` changes the current working directory 
to subdirectory `SUBDIR` assuming that subdirectory exists in the current 
working directory. The command `OPEN 1,8,15,"CD:←":CLOSE 1` (the `←` being 
the key in the upper left corner on the C64 keyboard) moves back to the 
parent directory. No idea why they did not pick the standard `..` instead 
of the rather obscure `←`. Next to `CD`, related advanced commands were 
added: `MD` to make a directory and `RD` to remove an (empty) directory.

Of course the C64 doesn't know about the `CD` command, but you as operator 
can issue them and the new directory feels like a new disk to the C64. 
Typically the SD2IEC and the Pi1541 also come with a file browser `FB64`. 
This browser does know the `CD` command and allows easy navigation through 
the file system on the SD card.


### Mounting a virtual floppy 

There is one more important concept. In the C64 retro community, floppy disks are ripped 
to a single file. They have the extension `.d64` and are useable as virtual disk in tools 
like VICE, the C64 Ultimate, and also the SD2IEC and Pi1541. 

SD2IEC and Pi1541 treat `.d64` files a bit like 
Windows treats `.zip` files. It is one _file_, but you can `CD` 
into it, and that "unzipped" `.d64` file is then the "mounted" _directory_. All 
high level commands (`LOAD`) work on the mounted floppy, and so do all 
advanced commands (`S0:MYGAME`). Even `OPEN 1,8,15,"CD:←":CLOSE 1` works;
it unmounts the `.d64` virtual floppy and switches to its containing directory.
Of course `OPEN 1,8,15,"CD:SUBDIR":CLOSE 1` does not work in the mounted `.d64` because
`.d64` files are supposed to be ripped 1541 disks, and the 1541 did not have a notion 
of subdirectories.


### Programming commands 

We should realize that the firmware (of SD2IEC and Pi1541) sees the high 
level and advanced commands come in via the IEC bus. They need to parse and 
understand them, and then execute them. That is the task for the software written 
by the programmers of those two devices. That is development work.

Unfortunately, next to the high level and advanced commands there is a third 
category, the _programming commands_. Chapter 8 of the 1541 
[user manual](https://www.zimmers.net/anonftp/pub/cbm/manuals/drives/1541_Users_Guide.pdf) 
introduces these with the text "The expert programmer can actually design routines 
that reside and operate on the disk controller". This is with commands such as `M-W` 
(memory write), `M-R` (memory read), and `M-E` (memory execute). Here is 
an example from the manual (recall that `RTS` has opcode 0x60 or 96 decimal).

> ![memory execute example](images/m-e-command.png)
>
> Example of programming commands from the 
> [user manual](https://www.zimmers.net/anonftp/pub/cbm/manuals/drives/1541_Users_Guide.pdf)

The above 1-byte "program" is written by the C64 in the memory of the 1541 drive 
(to addres 0x0300), and then the C64 gives a command to call 0x0300. The 1541 
starts executing there, finds opcode 96 (RTS), returns, and stops executing the 
program. I made a more elaborate example 
[blinky1541](https://github.com/maarten-pennings/C64howto/tree/main/blinky1541).
Anyhow, this 1541 feature is the essence of how fast loaders work.


### Browse mode versus emulation mode

So far SD2IEC and Pi1541 are very similar. Now we come to an important difference.
The SD2IEC does _not_ implement the programming commands, but Pi1541 _does_.
In a clever way.

The Pi1541 has two modes. The developer, Steve White, calls them _browse mode_ and
_emulation mode_. We could say that the SD2IEC only supports browse mode, and Pi1541 
supports both. When the Pi1541 starts, it is in browse mode. In browse mode, the 
firmware from Steve runs. This firmware implements the high level and advanced 
commands _including the `CD` command_. This allows all existing tools to 
browse all files on the SD card. As soon as a command comes in that `CD`s into 
a virtual floppy disk (I called this "mounting the floppy" above), the Pi1541 
switches to emulation mode. In emulation mode the Raspberry Pi _emulates the 1541_.

The emulation is serious. A real 1541 contains a 6502,
RAM, ROM, and VIAs. The Pi1541 emulates all of those, to the clock cycle. 
Timing of pulses on the IEC bus matters to the C64. What code does the emulated 
6502 run? It runs the original 1541 ROM from Commodore that you have to download 
and put on the SD card - Steve can't do that due to licensing reasons. As a consequence 
all `M-W` and `M-E` commands are working since they are implemented by that 1541 rom. 
Software that is uploaded and executed this way, by the fast loaders, also just works; 
it doesn't know the 1541 is emulated.

In emulation mode, an `OPEN 1,8,15,"CD:←":CLOSE 1` is intercepted, unmounts the `.d64`, 
sets the working directory to the parent, and switches back to browse mode. 

It is important for a user to know about these two modes, and to know which one is 
running when. For example, the advanced command `OPEN 1,8,15,"N0:DISKNAME,DN":CLOSE 1`
(recall `N` stands for `NEW` meaning "format") in _browse_ mode just _creates_ a 
new, empty, formatted virtual floppy image with the name `DISKNAME.d64` on the SD card. 
If you give the same command in emulation mode, the original 1541 firmware kicks in, 
and it will happily wipe (format) the entire mounted `.d64` virtual floppy.

> There is another, arguably more important, caveat to be aware of. 
> Writes to a d64 image (e.g. `SAVE "PYPROG"`) in emulation mode are not 
> written-through to SD card. All updates are in the Pi's RAM. 
> You _must unmount_ the virtual floppy (with `"CD:←"` or by pressing 
> the ESC key on the Pi1514). Only then the new content is written to 
> the FAT file system on the SD card.


### My additions

Mostly for fun, I have made a couple of additions to Steve's firmware. The `CD:subdir` command 
goes one subdirectory down, and `CD:←` goes one up. For some reason the `CD://` did not 
work. It is supposed to go to the root directory (yes with two slashes, something to 
do with multiple volumes). I fixed it. Secondly, I was missing a `pwd` command (print 
working directory). I added a `LOAD "$$"` command (I call it "sysinfo" that does that. 
I added both features in 
[v1.27](https://github.com/maarten-pennings/Pi1541#additions-in-this-fork).


## Real life operation

You might be wondering how the Pi1541 works in real life. 

You can operate the device completely from the C64:
you can do the `CD` commands combined with `LOAD "$"` by hand,
or let `FB64` do them for you.

But the Raspberry Pi can also be connected to a full keyboard and HDMI screen. 
The HDMI screen will always show the current directory and its contents (files and 
subdirectories), also when it is changed with `CD` (or `FB64` doing `CD`s).
It is also possible to change the current directory with a keyboard 
connected to the Raspberry Pi (via USB). 

There is a third option. We can connect an OLED and five buttons 
(Next, Prev, Select, Esc and Ins) to the Pi. 
The OLED will also always show the current directory 
(in sync with the HDMI screen), and the buttons also allow browsing 
(just like the USB keyboard).

Three methods, always in sync.

Oh, and in [v1.25](https://github.com/maarten-pennings/Pi1541#additions-in-this-fork)
I added support for a bigger (1.54") OLED, making it easier to read from a distance.

This table shows the functions of the five buttons. 
The swaplist is there to support large programs that use multiple floppies 
that need to be swapped during the run-time of the program.
When holding the INS key for a longer time, it acts as a shift key for the 
other four. It changes the device number as shown in the "Drive" column.

| Button | Description | Browse mode                  | Emulation mode       |Drive |
|:-------|:------------|:-----------------------------|:---------------------|:----:|
| SEL    | Select      | Change to sub dir (or mount) | -                    |    8 |
| PRV    | Previous    | Select previous dir          | Previous in swaplist |    9 |
| NXT    | Next        | Select next dir              | Next in swaplist     |   10 |
| ESC    | Escape      | Change to parent dir         | Unmount              |   11 |
| INS    | Insert      | Insert .d64 file in swaplist | -                    | shift|


## My design

I wanted a Pi1541 for my C64.
My goal was to make a device that resembles a real drive.

I wanted it fully featured: local keys and display, two IEC connectors 
(daisy chainable IEC like the real drives), access to SD card, reset button, 
power and activity LED, drive sound emulation, an LED to show data line status, 
access to HDMI out. 

I wanted it easy to operate: big buttons (Cherry keys), big display.
I wanted it to roughly have the same aspect ratios as the original 
drive (the 1541-II is 240×180×70 mm³). This posed a problem for the Cherry keys; 
they are relatively tall, so they need to be mounted on the "floor" PCB
that I was designing. 

My PCB would normally be the HAT ("Hardware at the Top") for the Raspberry Pi,
but due to the Cherry keys I needed to turn it upside down. My PCB is 
the HAB ("Hardware at the Bottom") with the Cherry keys and the IEC
connectors, and the Pi is on top, upside down. I did want an (OLED) display, 
so that needed to be even above the Pi.

This is a sketch of how I envisioned the entire system.

![Pi1541 concept sketch](images/pcbconcept.jpg)

From front to back we find:

- LEDs on the front.
- Next the Cherry keys on floor PCB.
- High rising pin header for a flipped OLED,  
  flipped in the sense that the top pixel row of the OLED is now facing the Cherry keys.
- After that we have the upside-down Raspberry Pi, SD card nicely accessible from the side.
- At the back we have the two IEC DIN connectors also on the floor PCB.
- Biggest problem is that the power (and HDMI) are not flush with the backside.
  I mitigated that by using an 
  [USB-C to micro USB converter](https://nl.aliexpress.com/item/1005009140380972.html), 
  which happens to bring the USB connector more to the back.

  ![USB extender](images/usbextender.jpg)


### Hardware

The simples solution Steve White offers is a Raspberry Pi and a level shifter
("Option A"). I based my design on [Option B](https://cbm-pi1541.firebaseapp.com/#:~:text=are%20all%20optional.-,Option%20B,-This%20option%20uses) of Steve.
This is for devices with two IECs. The Pi is not capable of sourcing enough current
for the extra "outgoing" IEC bus, so an extra hex tri-state inverter IC (7406) 
is used as bus driver. Steve warns that this driver IC needs to be selected carefully.

> [Ghosts'n Goblins Arcade](https://www.n0stalgia.org/common/pages/releases.php?op=showrelease&id=329) 
> can be used to test the capability of your hex inverter.
> If the title screen is corrupt then please substitute the hex inverter IC for a genuine branded version.

It was a bit of a puzzle how to connect the peripherals to the Pi.
The IEC wiring is different between Option A and B.
The local buttons, activity LED and buzzer are shown in Steve's schematics, but not the OLED.
I started with a breadboard.

![Breadboard](images/breadboard.jpg)

> The latest firmware (1.24) of Steve has a regression: the Activity LED is not working.
> I fixed that in my [fork](https://github.com/maarten-pennings/Pi1541).

I used the biggest 128×64 OLED module I had in stock.
It is [1.54 inch](https://nl.aliexpress.com/item/1005006579037427.html).
Since the various OLEDs modules tend to vary in the order of the four pins (SCL, SDA, VCC, GND), 
I added a second header (J3 and J4) and solder jumpers in between.
This allows me to solder patch wires if I have an OLED with different pin order.

I did add a (RESET) button on the IEC reset line. Pressing it resets the Pi1541 as well 
as the C64. Not sure if that is useful. I considered, but did not add 
a button that resets just the Pi; the Pi does have two pads for that.

I did add a beeper. 
I believe I have a [passive](https://www.circuitbasics.com/what-is-a-buzzer/).
These require a waveform to be generated by the micro controller (Pi in my case).
Active beepers are one-frequency-beepers, passive ones are more "speaker for audio".
Warning: the emulated drive sound is much lower quality then
what VICE offers; I believe it is basically a buzz when the drive head moves a track.

I added a red LED for power and green for the activity LED.
This is the color scheme used in the 1541-II. The original 1541 had the colors swapped.
My final addition was blue LED that is on the IEC data line.
We see it flicker when data is transmitted.

I ordered my PCBs at [JLCPCB](https://jlcpcb.com/DMP); 5 PCBs for €10.44 including 
shipping, arrived in 10 days.

![PCB front and back](images/pcb0.jpg)

A photo of the assembled PCB, including some 3D printed parts like the OLED holder
and standoffs.

![Assembled PCB](images/pcb1.jpg)

For a more extensive gallery and the design files itself, see directory [PCB](pcb).


### Case

See directory [case](case).


### Software

See other [repo](https://github.com/maarten-pennings/Pi1541).


## Links

- The official [documentation](https://cbm-pi1541.firebaseapp.com/) by Steve White.
- The GitHub [repo](https://github.com/pi1541/Pi1541) with the firmware from Steve White, 
  or my [fork](https://github.com/maarten-pennings/Pi1541).
- Older versions of [pi1541](https://cbm-pi1541.firebaseapp.com/whatsnew.html#oldversions.:~:text=the%20wrong%20folder.-,Old%20Versions,-.) firmware,
  or my [newer](https://github.com/maarten-pennings/Pi1541/releases).
- A Pi1541 [manual](https://www.combitronics.nl/download/pi1541%20Manual.pdf) by Michael Long.
- For commands (like `cd`) see the GitHub
  [repo](https://github.com/pi1541/Pi1541/blob/master/docs/IEC%20Commands.md) or
  the SD2IEC [user manual](https://c64os.com/post/sd2iecdocumentation) on C64OS.

(end)
