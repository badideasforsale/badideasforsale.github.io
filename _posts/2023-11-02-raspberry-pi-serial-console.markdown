---
layout: post
title:  "Raspberry Pi Serial Monitor"
date: 2023-11-02  10:30:00 -0700
tags: computers raspberry-pi serial
categories: computers
---

# Raspberry Pi Serial Monitor
I recently had to do some hardware troubleshooting for an intermittent fault. To do this, I used a Raspberry Pi as a serial monitor. It's pretty easy to do, but I'm writing it down here so I don't forget.

1. Get yourself a Raspberry Pi with a basic OS on it. [Raspbian](https://www.raspberrypi.com/software/) is good. I guess it's called Raspberry Pi OS now. 
1. I'm assuming you're doing a headless setup. Once the SD card is written, re-mount it on your computer and add a file called `ssh` to the root of the boot partition. This will enable SSH on the Pi.
1. Edit/create the `/etc/shadow` file. Add this line to set the `pi` user's password to `raspberry`':
```
pi:$6$SBgOl43F$QTRz0W27/786iJiN5YLlrsce7g3taQv8TiQYfcfBTXmwPs.jw5lOzu2ciZwHSFTaw16R.UaAr6ZR.ZRO6lWDR1
```
1. Plug the Pi in to your network and power it on.
1. SSH in to it, `ssh pi@ip-address`. The password is `raspberry`.
1. Change the password with `passwd`
1. Don't forget to change the password.
    * I will haunt your dreams if you don't change the password. Seriously. It's not that hard.
1. Install `cu` and `screen`. While `screen` has the native ability to access serial ports, if you want to disconnect from the Pi and leave the serial port open, `cu` is needed.
1. Look for the serial connection via `ls /dev/tty*`. It is probably `/dev/ttyUSB0`.
1. Start a new `screen` connection. Then start reading off the serial port with `cu -l /dev/ttyUSB0`.
1. There you go. Leave the Pi running, disconnect from screen, and you can come back at any time to see if the hardware malady you're trying to troubleshoot has revealed itself.

# References
* https://jasonmurray.org/posts/2020/raspconsole/