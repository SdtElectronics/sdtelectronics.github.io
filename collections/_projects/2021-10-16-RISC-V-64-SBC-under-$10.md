---
layout: article
title: Sub-10$ RISC-V 64 SBC
categories: Hardware
tags: linux evb risc-v
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2021-10-16-head-RISC-V-64-SBC-under-$10.jpg
---

An open-source single board computer under 10$ with a RISC-V 64 Core 

[Github repository](https://github.com/SdtElectronics/Xassette-Asterisk)

![front](https://github.com/SdtElectronics/Xassette-Asterisk/raw/master/img/front.jpg)

## Highlights
* Breaks out all IOs, involving analog peripherals, in a compact 56*56mm 2-layer board
* Comes with standard interfaces including USB, micro SD, LCD, Line-in and headphone
* Optimized components arrangement for soldering on a hot plate

## About the Chip
D1s/F133: RISC-V 64 single core @1.008G with in package 64MB DDR2

## Pin Out
![pinout](https://github.com/SdtElectronics/Xassette-Asterisk/raw/master/img/pinc.jpg)

Pins for LCD and DVP camera can also be used as IOs. See schematic below for detailed pin assignment.

## Schematic & BOM
![schematic](https://github.com/SdtElectronics/Xassette-Asterisk/raw/master/img/schematic.png)

The schematic in KiCAD format is available under [hw](https://github.com/SdtElectronics/Xassette-Asterisk/blob/master/hw). BOM in csv format is at [docs/BOM.csv](https://github.com/SdtElectronics/Xassette-Asterisk/blob/master/docs/BOM.csv). Do note that many components are optional (required by some specific peripherals)!

## Notes
* Leave all BOOT selection resistors unconnected if only one BOOT media is present
* Choose load capacitors according to specs of crystals
* When board is to be powered by 3.3V, connect to the power via the 3.3V pin of the pinheader, and `D4` should be soldered. Note USB host will not work properly in this condition due to the absence of 5V power.

## Accessories
### IO Expansion Board
To make use of IOs in the LCD port easier, [this expansion board](https://github.com/SdtElectronics/Xassette-Asterisk/blob/master/hw/auxiliary/Brk40p) converts all nets from FPC to 2.54mm pin headers with labeled IO indices. For 24pin DVP port, there is also an [expansion board](https://github.com/SdtElectronics/Biscuits/tree/master/24P_FPC_FFC_Breakout) but with no labels.

## FAQ
> Are you going to sell some manufactured boards?

No. I have no time and resource to batch manufacture this board. Some commercial boards should come in a couple of months (not from me).

> Where to buy some D1s chips?

The supply is not yet very sufficient, but it should be more available within a month (hopefully can be purchased directly from Allwinner). For now there are some suppliers providing samples on taobao.

> More information? Like what can this board do now?

The progress of this project is logged at [this Hackaday page](https://hackaday.io/project/182389-the-cheapest-risc-v-64-computer-by-now), and this repository will contain the source and documentation of this board only. Currently this board can boot up the tina Linux system (an OpenWRT fork by Allwinner) and populate a shell prompt via the serial, drive a parallel RGB display, and play sounds via the headphone socket. More functionalities will be tested in the future.

**Additional Words**

This project has gained unexpected popularity since the announcement. Thanks for all the interest! However, I am merely an enthusiast with limited time can be put on this, so I am sorry to disappoint who want to buy one. It is perfectly Okay if someone want to put this into production, as long as my work is acknowledged and the Licence is followed (better if you could contact me in advance). 

D1s is an awesome chip with many features to be exploited. Designing a PCB is not hard as the arrangement of pins is quite thoughtful. The crucial part is correct values of some key components, and they were all marked in the schematic. A symbol of D1s with annotated pins is also included in this repository, so this should also be a good start point for your own design.


## Licence
This project is available under the [CERN OHL-w v2](https://ohwr.org/project/cernohl/wikis/Documents/CERN-OHL-version-2) licence. 

## External Links
- [News on cnx-software](https://www.cnx-software.com/2021/10/30/open-source-hardware-allwinner-d1s-risc-v-linux-sbc/)
- [News on hackster.io](https://www.hackster.io/news/the-open-hardware-xassette-asterisk-gives-you-a-sub-10-linux-capable-risc-v-single-board-computer-115481c3ac5f)
- [News on electronics-lab](https://www.electronics-lab.com/xassette-asterisk-risc-v-64-sbc-features-allwinners-latest-d1s-soc-and-sells-for-less-than-10/)
- [Talk at RISC-V Open hours](https://www.linkedin.com/posts/drew-fustini-9732421b_risc-v-open-hours-2021-nov-3-activity-6861887437124321280-7i64/)
