---
layout: article
title: a Non-invasive Way to Investigate IO Connections
categories: Misc
tags: hacking Linux hardware
eyeCatcher: https://github.com/SdtElectronics/sun7i-std-dvr/raw/master/img/pinout.png
---
You got a TV set or a driving recorder by chance, and you found it carries a SoC running Linux. What more exciting is that SoC has good documentation and there are some IOs being routed out. You found the chance that something interesting can be made from this device, but you know nothing about how the pads on the board are connected to the chip. You have several options to investigate the connection, such as remove the chip and measure the connection among pads manually. You can even employ some fancy devices like a X-ray to reveal the internal routing of the board, but does this worth it?

However, there is a much more elegant way to do this if you can infer one of the connection and deduce the index of the corresponding IO. The idea is to connect that known IO to the pad under test, and change the levels of a range of IOs in the `sysfs` iteratively. If that known IO detects a change in the input level, this indicates that the last iterated IO is connected to that pad.

It's easy to write a shell script to automatize this process, and here I give this example for the standard `sysfs` interface in mainline kernel:
```bash
#!/bin/bash  

known=10

echo ${known} > /sys/class/gpio/export
echo in > /sys/class/gpio/gpio${known}/direction

cd /sys/class/gpio/
for i in $(seq 0 9) $(seq 11 17);
do   
	echo "${i}" > ./export
	echo out > "gpio${i}"/direction
	echo 1 > "gpio${i}"/value
	pre=($(cat ./gpio${known}/value))
	echo 0 > "gpio${i}"/value
	pos=($(cat ./gpio${known}/value))
	if [ ! $pre == $pos ]; then
		echo "gpio${i}"
		echo "${i}" > ./unexport
        exit
	fi
	echo "${i}" > ./unexport
done  
```
You have to modify the range appropriately to avoid changing the levels of some crucial pins e.g. IOs for mmc or flash.

Some version of Linux customized by the vendor may exhibit an interface for GPIOs different from the standard one, and you have to modify the script accordingly.