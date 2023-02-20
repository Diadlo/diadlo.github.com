# Photoframe reverse (part 1)

## Preface

About six months ago, during the summer of 2022, I came across [wrongbaud's
blog] which had a series of articles on firmware reverse engineering. As someone
who works with firmware and is interested in reverse engineering, I was
immediately drawn to these articles. In the series, I learned that there are
specialized tools available to connect to soldered flash and read its memory.

[wrongbaud's blog]: https://wrongbaud.github.io/

After purchasing those tools, I found myself stumped. I couldn't find anything I
wasn't afraid to break or ruin, so I put the tool away in a drawer.

## The Discovery

It all started when our digital photo frame broke. After powering it on, the
screen remained black, so I decided to buy a new one. And at this point I
already knew what I will do with the old one 😁.

And so, I disassembled the photo frame and found the main board, with all the
other components connected to it.

![Assembled digital frame](/img/frame_assembled.jpg)
![Disassembled digital frame](/img/frame_disassembled.jpg)
![Frame board](/img/frame_board.jpg)

Components from left to right connectors:

1. Some proxy-board to display
2. Buttons
3. Low-voltage differential signaling (LVDS) flat panel display (FPD)
4. IR sensor (photo frame has IR controller)
5. Speakers

Most interesting components on the board:

* Allwinner F1C200s - In the center of the board. A mobile ARM9 system-on-chip
  (SoC) with reach video and audio support. Good naming BTW 😁.
* Winbond 25Q64FVSIG - An SPI flash memory.

After reading an [article] about another person who had reverse engineered his
digital photo frame and found only JPEG default screen images, I wasn't sure
what to expect.

[article]: https://hackaday.com/2011/05/07/digital-picture-frame-reverse-engineering/

As wrongbaud tough us, I connected Test Clips to the flash memory and my
Raspberry Pi to make a dump. I was eager to see what I could uncover.

![Clips connected for flash reading](/img/frame_flash_reading.jpg)

Just for the reference. To read the flash I used the following command:

```
sudo ./flashrom -p linux_spi:dev=/dev/spidev0.0 -r ~/frame.rom -c W25Q64BV/W25Q64CV/W25Q64FV
```

This produced an 8MB file. I started with `binwalk` and `strings`. Getting ahead
of myself `binwalk` wasn't very useful but got me some clues.

With the initial analysis complete, I was ready to dive deeper into the firmware
and discover what was hidden inside.

## Flash analysis

Output from `binwalk`:

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

Examining the first few bytes of the flash, I found strings that are specific to
the Allwinner [EGON bootloader].

[EGON bootloader]: https://linux-sunxi.org/EGON

```
00000000  a8 00 00 ea 65 47 4f 4e  2e 42 54 30 98 f2 b8 6c  |....eGON.BT0...l|
00000010  00 3a 00 00 30 00 00 00  32 30 30 30 31 31 30 30  |.:..0...20001100|
00000020  31 32 30 30 31 31 30 30  53 55 4e 49 49 00 00 00  |12001100SUNII...|
00000030  78 02 00 00 31 32 30 30  00 00 00 80 00 00 00 00  |x...1200........|
```

The structure of the headers of the two bootloaders, eGON.BT0 and eGON.BT1, were
different from those described in articles I found, but I was able to reverse
engineer the correct structure thanks to debug strings.

![DRAM init code](/img/frame_dram_init.jpg)

I also found a mention of "MINFS" in the strings and was able to locate the
place where the system initializes. I saw the MINFS initialization and mount,
which uses a mount system with letters similar to Windows OS.

![OS start](/img/frame_os_start.jpg)

After that it load ["Melis"], Allwinner's in-house OS used for SoCs of F-series.

["Melis"]: https://en.wikipedia.org/wiki/Allwinner_Technology#F-Series

After some time in Google and Github I found [toolset] to modify firmware for
"Strit Fighter II Cabinet". This cabinet is based on the same chipset and file
system as my device. I took a look at the author github page and I found that
it's [wrongbaud]. The guy who inspired my to do all this research!

[toolset]: https://github.com/wrongbaud/sf-cabinet
[wrongbaud]: https://github.com/wrongbaud

Ok. Back to MINFS. The toolset includes a python script to extract the
filesystem. While going through the filesystem, I found numerous files with the
same [magic number] (AZ10) and detected by `binwalk` as LZMA compressed files,
but with unknown uncompressed sizes (which is mandatory for LZMA).

[magic number]: https://en.wikipedia.org/wiki/File_format#Magic_number

Luckily, I found a [HDD Guru forum page] where fzabkar user had the same issue
and was able to crack it. I wrote a simple tool to unpack the AZ10 files, and
finally, I was able to obtain the image of the welcome screen.

![Welcome image](/img/frame_welcome.jpg)

[HDD Guru forum page]: http://forum.hddguru.com/viewtopic.php?f=10&t=40760&start=20

One more thing I found in the dump was two identical FAT16 sections at the end.
These sections are likely the only changeable sections of the flash memory, as
they store user settings. The duplicate sections are most likely in place to
restore data in case of errors.

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
| 0x7e0000 |0x7f0000 | FAT16 (1)   |
| 0x7f0000 |0x800000 | FAT16 (2)   |

## Areas for Future Research

Despite successfully extracting the firmware and obtaining the welcome image,
the reason why the frame only shows a black screen remains unknown. At least
"welcome" image is in appropriate place 😉.

One way to approach this is to obtain logs from the SoC and analyze them to gain
insight into what might be causing the problem. Therefore, the plan is to
attempt to retrieve the logs and analyze them to help diagnose the issue.
