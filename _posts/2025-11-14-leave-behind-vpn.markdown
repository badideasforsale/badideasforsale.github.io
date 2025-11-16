---
layout: post
title:  "Leave Behind VPN"
date: 2025-11-14 10:30:00 -0700
tags: computers raspberry-pi vpn
categories: linux computers vpn
---


# Why?
Sometimes you want to be somewhere else. This isn’t to evade censorship; if you need that, this guide is not for you. The network traffic won’t be hidden, and Wireguard does not natively perform any kind of obfuscation. You can do some things, but deep packet inspection will probably point at your traffic and categorize it as a VPN.

It would be irresponsible to use this guide if using a VPN could cause users harm. Instead, this is for dealing with geo-blocks. Apps that only work if you are in a certain country, or content that is only available from certain IPs. Legal disclaimer: don't violate Terms of Services with this guide. That wouldn't be nice.

# Goal

By the end of this, we will have a Raspberry Pi that runs a Tailscale exit node. Tailscale is amazing, free, and extremely usable. The Pi will be pretty set-and-forget; while software updates happen, we can configure Tailscale to auto-update, and we can optionally set up the operating system to auto-update, too.

# Hardware
## Computery parts
A [Raspberry Pi 3B+](https://www.pishop.us/product/raspberry-pi-3-model-b-plus/) is the right choice. It has more than enough power to run Tailscale, has built-in Wifi and Ethernet, and crucially isn’t very picky about power. The 4 and 5 series Raspberry Pi boards demand more power and need a compatible power supply. The 3-series Pi’s will run off any USB port capable of charging an iPad. (2.1A and 5V is what it needs to be capable outputting.)

Obviously, you can use a different RPi if you like. I have a Pi 4 with a POE hat, but that device is doing more than just running a VPN. (And honestly, a 3 would probably have been fine for that purpose.) A Pi Zero 2W would be more than capable and fit in the power budget, but needs additional hardware if you're going to use Ethernet. Case selection is also more limited.

## Case
Oftentimes overlooked, but a decent case is an important choice. For this sort of appliance that’s probably sitting in a mate’s house, get one without a fan. I picked [this one off Amazon](https://www.amazon.com/dp/B0BJ6H7KJG). It is a simple hunk of aluminum. It doesn’t expose the GPIO pins, which means less dust or other crap can get inside. No fan for silent operation. It should be perfect.

## SD Card
I tend to use a 32GB card. MicroSD cards don’t have any sort of rated lifespan, so having a bigger card than strictly necessary means there’s more blocks to write to, which means less chance of wearing the card out by spreading the load around. Bigger is fine, but the cost difference is so small that going less than 32GB risks premature wear; and for something that is, by definition, not going to be close to you… it seems like the wrong time to try and save a buck or two.

# Software
I’ve been enjoying DietPi as the OS layer of my RPis. It’s a simple Debian-based distribution, supports a heap of single-board computers, and doesn’t do any sort of container fuckery.

A side rant: While I get why everyone uses containers, the whole container orchestration overhead for simple, small, projects is far too much. We’re not trying to squeeze every last bit of performance out of this. We’re going to follow the KISS principle: keep it simple, stupid. You want to run a goddamn kubernetes cluster? On a Raspberry Pi? So it can do one thing? You’ve added so much complexity it’s going to be ruinous. This post is about setting something up that can be literally left behind for years at a time with minimal to no maintenance. I don't want to think about Kubernetes.

So anyway, there's proof I'm old. Let’s get DietPi flashed onto an SD card.

## OS Installation
1. [Download DietPi](https://dietpi.com/#download)
1. [Download the Raspberry Pi Imager](https://www.raspberrypi.com/software/)
1. Insert the microSD card to your computer.
1. Click your way to victory.
    1. Choose the correct device (if you're following this guide, that's a Raspberry Pi 3)
    1. The option to use your own image is at the bottom of the `Choose OS` menu button. Select the DietPi file you download earlier.
    1. Choose the correct storage device, meaning your microSD card
1. Don't bother customizing the image. Just let it rip. 

Depending on how fast your hardware is, this will either take a short time or a not-so-short time. Once it's done, eject the card and stick it into your Pi.

## Connecting
Now we need to configure the Raspberry Pi to be a good VPN. There are two stages to this; the first one is connecting to the Pi, and the second is installing the software and tweaking some settings.

While not essential, I personally prefer to do the setup via SSH. You could connect a display and keyboard too, but regardless, the fastest and easiest mechanism is to hook the Pi up via an Ethernet cable to your home network.

Plug the Ethernet cable in first (optional: keyboard and display), then plug it in to power.

Via SSH, you'll need to find the IP address. There are ways to do this via your network's router, but connecting a display temporarily is an easy method. [Orion](https://orion.tube) turns an iPad into a portable monitor with the correct HDMI->USB-C adapter. Once you know the IP address (or have a keyboard/mouse connected) you're ready to connect.

The default credentials for DietPi are `root:dietpi`.

SSH to it and start the [DietPi setup process](https://dietpi.com/docs/install/). The first run will guide you through the initial setup, download some updates, and generally get things ready. The linked guide is a pretty simple one but will get you started.

You should restart once the initial setup is done. Run `reboot` at the command prompt. The SSH connection will break if that is how you are connected; go ahead and re-connect to it.

## Configuring
Now let's turn it into a VPN appliance. First things first, you need an [account with Tailscale](https://login.tailscale.com/start).
1. Log in or create your account: https://login.tailscale.com/start
1. Install Tailscale on whatever device you want to connect from; there are apps for [iOS](https://apps.apple.com/us/app/tailscale/id1470499037), [macOS](https://apps.apple.com/us/app/tailscale/id1475387142?mt=12), and [many more](https://tailscale.com/kb/1347/installation)

Now let's go back to the Raspberry Pi.

1. At a command prompt for your RPi, run `dietpi-software`.
1. Type `tailscale` and hit enter.
1. Use the spacebar to select Tailscale, then press enter.
1. Go back a level, navigate to Install, and hit enter. Tailscale will install.
1. Exit `dietpi-software` and go back to a normal prompt.
1. Run `tailscale up --advertise-exit-node`
1. Follow the instructions to link this device to your Tailscale account.
1. Make sure it _stays_ an exit node after reboots; run: `tailscale set --advertise-exit-node`
1. Lessen your workload. Run `tailscale set --auto-update` to ensure it stays updated.
1. Go to the Tailscale console.
    1. You will need to approve the Raspberry Pi as a new exit node.
    1. Disable key expiry for the Raspberry Pi and the device you plan to use to connect to it.

Disabling key expiry will keep things working. The default of having the keys expire is a safe one, but this isn't a security control we need to be concerned with for something that's personal-use-only.


DietPi settings, cron jobs

### DietPi settings 
These are some settings that will improve your project. They will either reduce wear on the microSD card, improve performance, or make your hair grow thicker and fuller.

First, let's add some cron jobs. Run `crontab -e` and add the lines below:
```
@reboot /usr/sbin/ethtool -K eth0 rx-gro-list off
@reboot /usr/sbin/ethtool -K eth0 rx-udp-gro-forwarding on
@weekly /boot/dietpi/dietpi-backup 1
```
The first two run on every reboot, and configure some networking settings that help Tailscale. I'll be honest here, `eth0` may be the Ethernet adapter only or just the first networking adapter. If you've doing this via WiFi-only, after configuring that, run `ifconfig` and make sure you know what network adapter the wireless connection is on. You may need to change `eth0` above to `wlan0` or `eth1`.

The third line runs a weekly backup. DietPi only will run daily backups, or not at all. Weekly is plenty. Not at all is also probably fine, if we're honest with ourselves. Untested backups are just wishful thoughts.

Next up: `dietpi-config`. DietPi has a nice terminal user interface for setting various options. At a command line, run `dietpi-config`.

* Display Options
    * If you are really trying to save every last milliwatt of power, you can disable the LEDs on the board here. Not really worth it for this project, since we'll have mains power.
    * Ensure `GPU/CPU memory split` is set to `16 : Server/Console/Headless`, or whichever the lowest value is. 
* Audio Options
    * There is nothing to configure in here.
* Performance Options
    * I don't recommend modifying anything in here. 
* Advanced Options
    * APT
        * Set the `APT cache` to be `In RAM`
        * Set the `APT lists` to be `In RAM`
        * Set `List Compression` to `On`
        * Leave `APT archives` set to `On disk`
    * Time sync mode
        * Change it to `3 : Boot + Hourly`. I was getting minor drift when it was `2 : Boot + Daily (Recommended)` which wasn't enough to be a problem, but was enough to trigger an alert in my monitoring tools which annoyed me.
    * Serial/UART
        * Ensure it is off
    * Bluetooth
        * Set to off
    * I2C State
        * Ensure it is off
* Language/Regional Options
    * Locale should be `[C.UTF-8]`
    * Timezone should be the timezone you are going to be leaving this appliance behind in. 
    * Keyboard should be `gb` or `us`, unless you're using a different keyboard, in which case, set it there.
* Security Options
    * You can change the passwords here if you have regrets
    * The Hostname will be the name that appears in Tailscale's console. You don't have to change it, but nobody will complain if you do.
* Network Options: Adapters
    * If you are using WiFi, configure it here
    * If you are using Ethernet, disable WiFi here
    * You can leave `IPv6` on.
* Network Options: Misc
    * I strongly recommend leaving `Boot wait for network` set to `On`
* AutoStart Options
    * Nothing to configure in here. Tailscale is configured to start with the system elsewhere.
* Tools
    * Nothing to do here.

Optional: Run `dietpi-banner` and choose what you want to show on login. LAN and WAN IPs are good choices. A quick link to the [DietPi Documentation for System Configuration](https://dietpi.com/docs/dietpi_tools/system_configuration/) page.

### WiFi
Ideally this thing will be hardwired. So much less to go wrong with cables. But if you have to, it does work fine via WiFi. (I can confirm streaming 1080p works, which means the Pi’s WiFi has to do “double duty”, taking in and transmitting with the associated extra CPU usage and limited bandwidth.) Also, wifi does use a tiny bit of power, so disabling it will help the Pi run cooler and not consume quite so much power. It just sips power anyway, but every bit helps.

You can configure a watchdog, so if the Pi doesn’t see the WiFi access point after some period of time, it will reboot. Because turning it off and back on again fixes lots of problems, and while we could ask our mates to unplug and plug it back in, this is a safer way to make it all work without having to bother the people graciously putting this on their network.

If you want a WiFi watchdog, follow these steps. It's unlikely you will need to do this with a hardwired system. `apt-get install watchdog` will install the tool, then edit the configuration via `nano /etc/watchdog.conf`. You will need to uncomment the `ping` directive and give it the IP address of the router on the network. If you get this wrong, the system will end up constantly rebooting, so be careful! If your home/setup network differs from the network you'll be leaving this behind at, you could end up stuck in a reboot loop!

I have the following settings:
```
ping = 192.168.9.1
interface = wlan0
interval = 30 # seconds
```

If the wifi connection is lost for more than a minute, the Raspberry Pi will automatically reboot itself in an attempt to repair the connection. This is why Ethernet is better; less stuff to go wrong.

# Testing
Okay, let's see if all this work has worked out. Restart the device by running `reboot` at the command line, count to 45, and see if it shows up as online in the Tailscale console.

If it does, try connecting to it from your device. You'll connect to Tailscale, then select the Raspberry Pi as the exit node. If that works, it's a good sign this will work in the future. Disconnect for now.

Next, leave it running for a couple days. Connect while out and about from a mobile phone and see if it works. If that works, it's ready to be left behind on a friendly network for you to connect into.

# Optional
SSH is made easier when you don't have to care about keys. Your hardware device just does it. I wrote a post awhile back about [using Secretive for SSH keys](https://brandonsherman.com/computers/2023/11/03/using-secretive-for-git.html).

# Conclusion
This is the sort of thing you can easily do, then box up and mail to someone you trust to plug a couple wires in. It’ll just work once connected to power, and then your packets can come from somewhere else.