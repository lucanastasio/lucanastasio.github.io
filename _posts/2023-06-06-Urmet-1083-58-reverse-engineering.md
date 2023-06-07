---
layout: post
title:  "Urmet 1083/58 intercom call forwarding device reverse engineering"
date:   2023-06-06 10:28:03 +0100
categories: misc
---

![Urmet 1083/58](/assets/img/rev_eng/urmet_1083_58/urmet_1083_58.png)

I live in an apartment, unfortunately my video doorbell can't be replaced since the system (Urmet 2Voice) is shared among all the apartments in the building. I found out that the vendor has a dedicated call forwarding device, "That's great!" I thought, but turns out it wasn't that good. The overall system is really cumbersome and unintuitive, the mobile app is almost unusable (don't trust me, read the reviews yourself, both [Android](https://play.google.com/store/apps/details?id=it.urmet.callme&pli=1) and [iOS](https://apps.apple.com/it/app/urmet-callme/id1103676853)) and crashes randomly, there is no integration with third parties whatsoever.

I have already arranged a hollow in every room of the apartment for a room-control tablet/kiosk using a Raspberry Pi with its 7" display. I would really like to get rid of the current intercom and use the tablets instead. Getting in touch with the customer service was another dead end, as expected, so here we are, trying to get something useful out of this thing.

The call forwarding device itself, like many of their intercoms, is split into two different boards: an interface board betweeen the bus and actual user signals (see below) and a user interface like a handset and a display or a processor board like in this case. The following images have a high resolution.

# Interface board

Size: 83x62 mm

![Urmet 1083/58 interface board front](/assets/img/rev_eng/urmet_1083_58/urmet_1083_58_interface_front.jpg)

![Urmet 1083/58 interface board back](/assets/img/rev_eng/urmet_1083_58/urmet_1083_58_interface_back.jpg)


I'll just write a brief description of the interface board since the major area of intervention is the processor board instead. This board is mostly analog, it has just a microcontroller ([Freescale Kinetis KL05](https://www.nxp.com/docs/en/data-sheet/KL05P48M48SF1.pdf)) for digital control purposes and bus address setting with a DIP switch. The analog part is introduced below as part of the bus. The signals with the processor board are exchanged through optocouplers and transformers.

### 2Voice introduction

The "bus" itself is based on a 100 Ohm twisted pair carrying at the same time power, control data (digital), video and bidirectional audio (both analog). While everything else requires a narrow bandwidth and thus doesn't need a line termination, the video part, although being likely FM self-modulated (to avoid interaction with the other signals on the bus?), needs a proper line termination. To accommodate the shared nature of the bus and its ramifications, a relay is employed to connect a 100 Ohm (reactive, apparently) termination when needed. The video signal then passes through a transformer and is amplified (from differential to single-ended), then self-demodulation takes place using XOR logic ports and finally the recovered signal is low-pass filtered. Goes without saying that only one device at a time can receive the video signal.

### Board to board connector

There is a 2x10 pin, 2.54 mm pitch connector between the boards, marked J4 on both, however pin 1 is marked only on the processor board.

|Pin|Int. side conn.|Proc. side conn.|Function              |
|---|---------------|----------------|----------------------|
|1  |               |3.3V regulator  |                      |
|2  |               |12V regulator   |                      |
|3  |               |                |`chiamata` in         |
|4  |               |                |`personality[0]` (msb)|
|5  |               |                |`linealibera` in      |
|6  |               |                |`personality[1]`      |
|7  |               |                |`allarme` in          |
|8  |               |                |`personality[2]`      |
|9  |               |                |`video` out           |
|10 |               |                |`personality[3]` (lsb)|
|11 |               |                |`gancio` out          |
|12 |               |ML7037 input¹   |Audio in              |
|13 |               |                |`carraia` out         |
|14 |               |ML7037 output²  |Audio out             |
|15 |               |                |`pedonale` out        |
|16 |               |GND             |Common ground         |
|17 |               |TVP5150 input³  |Video in              |
|18 |               |JP2             |Bus power+            |
|19 |               |GND             |Common ground         |
|20 |               |JP3             |Bus power-            |

*Notes:*
1. Through gain setting resistors and integrated amplifier.
2. Pin 9, through 1 kOhm resistor.
3. Through an LC low pass filter and a decoupling capacitor.

# Processor board

Size: 83x70 mm

![Urmet 1083/58 processor board front](/assets/img/rev_eng/urmet_1083_58/urmet_1083_58_processor_front.jpg)

![Urmet 1083/58 processor board back](/assets/img/rev_eng/urmet_1083_58/urmet_1083_58_processor_back.jpg)

The processor board transmits and receives digital control signals and analog audio, while also decoding analog video. Network connectivity is provided by WiFi and Ethernet interfaces. DC power can be either sourced from the bus or a separate external power supply.

It's based on a chipset similar to the final product in many ways ([see here for someone who already had to deal with that](https://elinux.org/images/3/3c/Ceresoli-terrible-bsp-elce2017.pdf)) in terms of capability to get the user frustrated.

### Hardware

Click to see the datasheet:
- SoC: [Nuvoton N32926U1DN](https://datasheet.lcsc.com/lcsc/1810010117_Nuvoton-Tech-N32926U1DN_C93158.pdf) (ARM926EJ-S CPU @ 240 MHz, 64 MB DDR2)
- Flash: [Zentel A5U1GA31ATS-BC](https://www.zentel-europe.com/datasheets/A5U1GA341ATS(BF)_v1.4_Zentel.pdf) (128 MB NAND)
- Audio codec: [OKI/Lapis ML7037-003](https://www.lapis-tech.com/en/data/datasheet-file_db/telecom/FEDL7037-003-03.pdf)
- Video decoder: [TI TVP5150AM1](https://www.ti.com/lit/ds/symlink/tvp5150am1.pdf)
- WiFi: [Realtek RTL8188CTV](https://pdf1.alldatasheet.com/datasheet-pdf/download/1572381/ETC/RTL8188CTV.html)
- Ethernet PHY: [Realtek RTL8201F](https://datasheet.lcsc.com/lcsc/1809191828_Realtek-Semicon-RTL8201F-VB-CG_C45044.pdf)
- EEPROM: [Microchip 93LC46BI](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/20001749K.pdf)
- Main regulator: [TI TPS54060](https://www.ti.com/lit/ds/symlink/tps54060.pdf)

#### JP1 - JTAG

There is an unsoldered edge connector marked JP1, tracing the signals reveals the following connections, thus it is most likely a JTAG connector (alternate pin functions can be found in the datasheet).

|Pin|SoC pin|Function   |
|---|-------|-----------|
|1  |-      |3.3V       |
|2  |77     |TRST_a     |
|3  |79     |TDI_a      |
|4  |80     |TMS_a      |
|5  |81     |TCK_a      |
|6  |R100   |?          |
|7  |78     |TDO_a      |
|8  |R99    |?          |
|9  |-      |GND        |

#### J2 - micro SD

There is an unpopulated footprint for a micro SD socket corresponding to the [Molex 105162-0001](https://www.molex.com/content/dam/molex/molex-dot-com/products/automated/en-us/salesdrawingpdf/105/105162/1051620001_sd.pdf). Since the SoC's boot order is SD0, NAND0, NAND1, SD1, SD2, soldering the appropriate socket and inserting a micro SD card should make it boot from it first without having to modify the NAND flash on board or anything else. Unlike other SoCs in the family this one doesn't support SPI flash boot.

|Pin|SoC pin|Function   |
|---|-------|-----------|
|1  |1      |SD0_D2     |
|2  |2      |SD0_D3     |
|3  |3      |SD0_CMD    |
|4  |-      |3.3V       |
|5  |4      |SD0_CLK    |
|6  |-      |GND        |
|7  |5      |SD0_D0     |
|8  |6      |SD0_D1     |

#### 2x10 header

The header footprint on the back side seems to have multiple functions for programming, debugging and testing purposes. To enter recovery mode, where the processor becomes programmable through USB attached to an host computer, one has to tie the USB detect and recovery mode pins to 3.3V, after entering recovery mode the latter has to be disconnected in order to be able to program the NAND flash on board. The tool can be found [here](https://github.com/OpenNuvoton/N32905_NonOS_Tool/tree/master/TurboWriter/TurboWriter%20V2.30.003_N329x6) and the manual [here](https://github.com/OpenNuvoton/N32905_NonOS_Tool/blob/master/TurboWriter/TurboWriter%20Tool%20User%20Guide.pdf).

Attaching a terminal to the serial pins normally shows a rather useless log from the boot ROM only. The kernel won't be connected to the serial port unless the test mode pin is tied low. Doing so not only reveals the kernel log but also enables the interactive shell, with the `screen` program at 115200 baud works flawlessly.

|Pin|SoC pin|Function   |
|---|-------|-----------|
|1  |85     |Test mode**|
|2  |-      |3.3V       |
|3  |?      |Debug mode?|
|4  |104    |UART TX    |
|5  |9      |USB Detect*|
|6  |105    |UART RX    |
|7  |29     |USB D- (W) |
|8  |107*** |Recovery*  |
|9  |30     |USB D+ (G) |
|10 |-      |GND        |

\* Active high.

\** Active low.

\*** CHIPCFG[0] is active low but pin 8 is connected to it through a PDTC143ET transistor and corresponds to the NAND flash CS0 pin.

### Firmware

As anticipated, the software support from the SoC vendor is no joy. The device manufacturer also customized the NAND flash bootloader and the kernel further. All you are left with are those repositories:
- [Baremetal BSP](https://github.com/OpenNuvoton/N32926_NonOS_BSP)
- [Linux BSP](https://github.com/OpenNuvoton/N32926_Linux_BSP)
- [Linux applications](https://github.com/OpenNuvoton/N32926_Linux_Applications)

#### Flash memory layout

The flashing tool manual explains the non-volatile memory layout, while binwalk reveals the following information from the NAND flash dump:

|Decimal |Hex      |Block|Size|Description                                                                                                                 |
|--------|---------|-----|----|----------------------------------------------------------------------------------------------------------------------------|
|651264  |0x9F000  |318  |    |ASCII cpio archive (SVR4 with no CRC), file name: "/dev", file name length: "0x00000005", file size: "0x00000000"           |
|651380  |0x9F074  |     |    |ASCII cpio archive (SVR4 with no CRC), file name: "/dev/console", file name length: "0x0000000D", file size: "0x00000000"   |
|651504  |0x9F0F0  |     |    |ASCII cpio archive (SVR4 with no CRC), file name: "/root", file name length: "0x00000006", file size: "0x00000000"          |
|651620  |0x9F164  |     |    |ASCII cpio archive (SVR4 with no CRC), file name: "TRAILER!!!", file name length: "0x0000000B", file size: "0x00000000"     |
|971789  |0xED40D  |     |    |Certificate in DER format (x509 v3), header length: 4, sequence length: 1312                                                |
|1901917 |0x1D055D |     |    |Certificate in DER format (x509 v3), header length: 4, sequence length: 1284                                                |
|1902045 |0x1D05DD |     |    |Certificate in DER format (x509 v3), header length: 4, sequence length: 1288                                                |
|2519765 |0x2672D5 |     |    |Certificate in DER format (x509 v3), header length: 4, sequence length: 5500                                                |
|2522689 |0x267E41 |     |    |Certificate in DER format (x509 v3), header length: 4, sequence length: 9620                                                |
|3425104 |0x344350 |     |    |Linux kernel version 2.6.35                                                                                                 |
|3471204 |0x34F764 |     |    |CRC32 polynomial table, little endian                                                                                       |
|3860425 |0x3AE7C9 |     |    |Unix path: /home/daniele/Tmp/1083_58/4.4.0-3/buildroot/output/build/linux-custom/arch/arm/include/asm/dma-mapping.h         |
|4200847 |0x40198F |     |    |LZMA compressed data, properties: 0xC0, dictionary size: 0 bytes, uncompressed size: 32 bytes                               |
|4237239 |0x40A7B7 |     |    |Intel x86 or x64 microcode, pf_mask 0x8000, 2000-01-20, size 18882560                                                       |
|17301504|0x1080000|8448 |    |CramFS filesystem, little endian, size: 18309120, version 2, sorted_dirs, CRC 0x97AFA656, edition 0, 9119 blocks, 2001 files|
|84410368|0x5080000|41216|    |UBI erase count header, version: 1, EC: 0x2, VID header offset: 0x800, data offset: 0x1000                                  |

Probably it didn't get everything correctly, but the filesystems were correct. However before understanding how to enter the test mode I already took the hard road and desoldered the NAND flash to read it. I used a NAND Lite tool which dumped the flash blocks (2 kB) along with the ECC data (64 B), thus I had to remove the latter using the following (trivial) python script.

{% highlight python %}
import sys

if __name__ == '__main__':
    f_in = open(sys.argv[1], 'rb')
    f_out = open(sys.argv[2], 'wb')
    block_size = int(sys.argv[3])
    ecc_size = int(sys.argv[4])
    print("Block size {}, ecc size {}".format(block_size, ecc_size))
    l_in = len(f_in.read())
    f_in.seek(0)
    block_count = int(l_in/(block_size+ecc_size))
    print("Input size {}, block count {}".format(l_in, block_count))
    for i in range(block_count):
        f_out.write(f_in.read(block_size))
        _ = f_in.read(ecc_size)
    f_in.close()
    print("Output size {}".format(f_out.tell()))
    f_out.close()
{% endhighlight %}

Binwalk found data starting at block number 318. The CramFS filesystem starts at flash block 8448 and the UBI filesystem starts at 41216. Binwalk wasn't able to extract the CramFS filesystem, so I had to do that using cramfsck. The UBI filesystem is mounted after boot by an init script, it's used to store the main application configuration file. If it doesn't exist then the script will format the partition and create a 40 MiB filesystem. Here is the partition "table" from `/proc/mtd`:

|Dev |Size               |Erasesize|Name      |
|----|-------------------|---------|----------|
|mtd0|00080000 (512 kiB) |00020000 |"SYSTEM"  |
|mtd1|00800000 (8 MiB)   |00020000 |"KERNEL-A"|
|mtd2|00800000 (8 MiB)   |00020000 |"KERNEL-B"|
|mtd3|02000000 (32 MiB)  |00020000 |"ROOTFS-A"|
|mtd4|02000000 (32 MiB)  |00020000 |"ROOTFS-B"|
|mtd5|02f80000 (47.5 MiB)|00020000 |"DATA"    |

#### Bootloader and kernel

As it turned out, the bootloader passes a few command line arguments to the kernel according to the following string extracted from the dump. It reads the GPIOs that enable test and debug modes and passes their values as custom arguments. It's not clear why the MTD partition number is also passed as a variable argument, but digging through some more strings from the bootloader there seems to be some sort of partition table in the external EEPROM.

```
mem=64M noinitrd console=ttyS1,115200n8 root=/dev/mtdblock%d ro rootfstype=cramfs rdinit=/sbin/init u_test_mode=%d u_debug_mode=%d
```
The MTD partition number for the CramFS root seems to be 3.
Actually, until block 318 some data repeats itself, like there are multiple bootloader copies.

The kernel itself seems to occupy only 46100 bytes according to binwalk, but more likely it starts at block 318 ends just before the CramFS image, thus it should be ~16 MB. It implements a custom sysfs interface for some board features, the following entries can be found in `/sys/class/urmet/`:

|Name         |R/W|Description                          |
|-------------|---|-------------------------------------|
|allarme      |R  | |
|allin        |R  |All 10 input pins values             |
|carraia      |R/W| |
|chiamata     |R  | |
|debugm       |R  |Debug mode enabled                   |
|efw          |   | |
|emac         |   | |
|eokituning   |   | |
|eroot        |   | |
|etest        |   | |
|eth          |R/W|Ethernet PHY enable/disable          |
|eunlock      |   | |
|gancio       |R/W| |
|hwrev        |   | |
|ledg         |R/W|LED green component                  |
|ledr         |R/W|LED red component                    |
|linealibera  |R  | |
|pb1          |R  |Pushbutton state, 1 when pressed     |
|pedonale     |R/W| |
|personality  |R  |Interface board type                 |
|testm        |R  |Test mode enabled                    |
|video        |R/W|Enable intercom video?               |
|wifi         |R/W|WiFi enable/disable                  |

The personality is encoded by 4 input pins, in my case the decimal value corresponds to 14, the recognized values are:

|Dec|Bin |Description|
|---|----|-----------|
|4  |0100|1722_85    |
|8  |1000|4N_AUDIO   |
|12 |1100|4N         |
|13 |1101|1723_73    |
|14 |1110|2VOICE     |
|15 |1111|Unconnected|

Some out-of-tree modules provide drivers for WiFi and audio.

#### Main application

In normal mode, the init script starts the main application. In test mode however if a `/mnt/data/autorun.sh` file exists in the UBI FS starts this one instead, otherwise it does nothing else. Telnet is only started in test mode.
The main application seems to be based on linphone, acts as a SIP client (there is a `/usr/local/urmet/linphonerc` file). The following is the configuration file stored in the UBI partition as `/mnt/data/config.json`, the login is composed of the string `cfw` (standing for call forwarding?) followed by a 12 digit numeric code corresponding to my username, the password is encoded like an UUID.

{% highlight json %}
{
        "action" : 1,
        "sipserver" : "sip.urmet.com:6060",
        "login" : "cfw************",
        "password" : "********-****-****-****-************",
        "stunserver" : "sip.urmet.com",
        "videoquality" : 2,
        "name" : "dispositivo_test",
        "alarm" : "ALRM_",
        "dhcp" : true,
        "eth" : true,
        "ip" : "",
        "gateway" : "",
        "mask" : "",
        "dns" : "",
        "wifissid" : "",
        "wifipassword" : "",
        "wifikey" : "",
        "wifissidlist" :
        [
        ],
        "wifikeylist" :
        [
        ],
        "wifiqualitylist" :
        [
        ],
        "nowifibeginlist" :
        [
        ],
        "nowifiendlist" :
        [
        ],
        "timezone" : "Europe/Rome",
        "profileLowResolution" : "qqvga",
        "profileLowFrameRate" : 12,
        "profileLowKbps" : 128,
        "profileMediumResolution" : "qvga",
        "profileMediumFrameRate" : 12,
        "profileMediumKbps" : 300,
        "profileHighResolution" : "qvga",
        "profileHighFrameRate" : 30,
        "profileHighKbps" : 1024
}
{% endhighlight %}

#### Network

On boot, the ethernet port is not started automatically despite an init network script exists. To do so there is another script, executing `/usr/local/urmet/scripts/urmet_net.sh eth` brings the Ethernet PHY up, then executing `udhcpc` will get an IP address. In normal mode seems that the networking part is still managed by the main application, it reads the configuration file to determine which interface to use.

## Home assistant integration

To be continued...








