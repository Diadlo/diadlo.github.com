# Photoframe reverse (part 1)

## Preface

Half of a year ago (summer 2022) I faced [wrongbaud's blog]
with a series of articles about firmware reverse engineering. I work with
firmware myself, and also I'm keen of reverse engineering. That's why this
articles catched me. In this series I read, that there are special tools to
connect to soldered flash and read it's memeory.

[wrongbaud's blog]: https://wrongbaud.github.io/

I bough one and started thinking where I can apply this technique. I didn't find
anything that I wasn't afraid to broke and put this away in a drawer.

## Internals overview

Peceinely our digital photoframe broke. After power off I can see that
screen is powered off but it was black. Then I bought a new photo fram, because
I knew what I will do with the old one 😁.

I disassembled it and found main board:

![Assembled digital frame](/img/frame_assembled.jpg)
![Disassembled digital frame](/img/frame_disassembled.jpg)
![Frame board](/img/frame_board.jpg)

All other components are connected to it (from left to right connectors):

1. Some proxy-board to display
2. Buttons
3. Low-voltage differential signaling (LVDS) flat panel display (FPD)
4. IR sensor (photo frame has IR controller)
5. Speakers

Most interesting components on the board:

* Allwinner F1C200s - In the center of the board. Mobile ARM9 SoC with reach
  video and audio support. Good naming BTW 😁.
* Winbond 25Q64FVSIG - SPI flash memory.

Previously I read article ["Digital picture frame reverse engineering"] where
one guy also read flash memory of his digital photo frame. But he found only
JPEG default screen images (at least he didn't cover details if he found
something else). So I didn't expect much, but this still was very interesting
for me.

["Digital picture frame reverse engineering"]: https://hackaday.com/2011/05/07/digital-picture-frame-reverse-engineering/

As wrongbaud tough us, I connected Test Clips to flash and to my Raspberry PI
to made a dump.

![Clips connected for flash reading](/img/frame_flash_reading.jpg)

Just for the reference. To read flash I used this command:

```
sudo ./flashrom -p linux_spi:dev=/dev/spidev0.0 -r ~/frame.rom -c W25Q64BV/W25Q64CV/W25Q64FV
```

Now I had 8MB file to analyze. I started with binwalk and strings.
Getting ahead of myself binwalk wasn't very useful but got me some clues.

## Flash analysis

Output from binwalk:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
616380        0x967BC         LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, missing uncompressed size
1246876       0x13069C        LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, missing uncompressed size
1401995       0x15648B        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 4bit
1406219       0x15750B        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 4bit
1406411       0x1575CB        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 4bit
1410059       0x15840B        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 4bit
1411979       0x158B8B        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 4bit
1994224       0x1E6DF0        PC bitmap, Windows 3.x format,, 380 x 24 x 32
5823972       0x58DDE4        LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, missing uncompressed size
7016032       0x6B0E60        Boot section Start 0x48124812 End 0x4007EFE
7915609       0x78C859        bix header, header size: 64 bytes, header CRC: 0x4C414D45, created: 1997-03-18 06:37:45, image size: 539517541 bytes, Data Address: 0x74612955, Entry Point: 0x55555555, data CRC: 0x55555555, image name: "UUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUUU"
```

And I found lots of strings in the flash. Now I was sure that there is not only
images.

In the first bytes flash has multiple strings. I checked them and found
Allwinner-specific [EGON bootloader]. Actually the whole site is very
interesting. It's an open source community around Allwinner SoC. Unfortunately,
it has not so many information about F-Series SoCs.

[EGON bootloader]: https://linux-sunxi.org/EGON

```
00000000  a8 00 00 ea 65 47 4f 4e  2e 42 54 30 98 f2 b8 6c  |....eGON.BT0...l|
00000010  00 3a 00 00 30 00 00 00  32 30 30 30 31 31 30 30  |.:..0...20001100|
00000020  31 32 30 30 31 31 30 30  53 55 4e 49 49 00 00 00  |12001100SUNII...|
00000030  78 02 00 00 31 32 30 30  00 00 00 80 00 00 00 00  |x...1200........|
```

Basically in the beginning it's ARMv5TE executable code of two bootloaders
(eGON.BT0 and eGON.BT1). The structure of headers a bit different that provided
in article. But thanks to debug strings it was quite easy to reverse the right
structure.

![DRAM init code](/img/frame_dram_init.jpg)

In strings I found MINFS mention and tried to found it in the internet and in
the code. Then I found place where system initializes. Here we can see MINFS
initialization and mount (BTW it uses mount system with letter like Windows OS).

![OS start](/img/frame_os_start.jpg)

After that it load "Melis" and jumps to kernel. And we can found that:

> Allwinner's in-house operating system Melis2.0, which is now mainly used in
> vehicle multimedia systems, E-ink readers, video intercom systems, and so on

in [wikipedia](https://en.wikipedia.org/wiki/Allwinner_Technology#F-Series)
and in [linux-sunxi wiki](https://linux-sunxi.org/Allwinner_SoC_Family#.22F.22-Series).

After some time in Google and Github I found [toolset] to modify firmware for
"Strit Fighter II Cabinet". The idea is that the cabinet is based on the same
chipset and FS. I took a look at the author github page and I found that it's
[wrongbaud]. The guy who inspired my to do all this research!

[toolset]: https://github.com/wrongbaud/sf-cabinet
[wrongbaud]: https://github.com/wrongbaud

Ok. Back to MINFS. This toolset has python script to extract filesystem. In this
FS I found many files that binwalk detects as LZMA compressed files but with
unknown original size (it's mandatory field of LZMA header). Especially there
was many files with the same [magic number (signature)] "AZ10".

During research I found a [HDD Guru forum page] where user fzabkar also faced
the same issue and cracked it. I wrote simple tool to unpack AZ10 files and here
we go:

![Welcome image](/img/frame_welcome.jpg)

Finally I got image of welcome screen.

[magic number]: https://en.wikipedia.org/wiki/File_format#Magic_number
[HDD Guru forum page]: http://forum.hddguru.com/viewtopic.php?f=10&t=40760&start=20

Also, I forgot to mention that in the end of dump I found two completely the
same FAT16 sections. Probably they are the only changable sections of the flash
since they store user settings. And then probably they have duplicate to restore
data in case of errors.

## Flash structure

Finally we got this structure of flash:

| Start    | End     | Description |
|----------|---------|-------------|
| 0x000000 |0x0002a8 | eGON.BT0    |
| 0x0002a8 |0x0039ac | Code boot 0 |
| 0x0039ac |0x006000 | 0xFFFF      |
| 0x006000 |0x00e2b0 | eGON.BT1    |
| 0x00e2b0 |0x017440 | Code boot 1 |
| 0x017440 |0x024000 | 0xFFFF      |
| 0x024000 |0x795640 | MINFS       |
| 0x795640 |0x7e0000 | 0x0000      |
| 0x7e0000 |0x7f0000 | FAT16 (x1)  |
| 0x7f0000 |0x800000 | FAT16 (x2)  |

## Future research

We still don't know why frame shows only black screen. At least "welcome" image
is in appropriate place 😉.

Later I'm going to try to obtain logs from SoC and analyze them.
