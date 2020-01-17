# Advanced Wi-Fi Attacks Using Commodity Hardware

We provide tools to perform low-layer attacks such as reactive and constant jamming using commodity devices. Reactive jamming allows you to block specific Wi-Fi packets. For example, all beacons and probe responses of a specific Access Point (AP) can be jammed. It has been tested with the following devices:

* [AWUS036NHA](http://www.amazon.com/dp/B004Y6MIXS?tag=modwiffir-20)
* [WNDA3200](http://www.amazon.co.uk/dp/B009XSPZ0U?tag=modwiffir-20)
* [Azurewave AW-NU138](https://wikidevi.com/wiki/AzureWave_AW-NU138). Confirmed to work by a user. It's a mini size type without external connector.
* [TP-Link WN722N v1](https://wikidevi.com/wiki/TP-LINK_TL-WN722N). Note that the newer v2 version is not compatible with ModWifi because it no longer uses an Atheros chip. Be sure to buy the v1 version.

This work was the result of the paper [Advanced Wi-Fi Attacks Using Commodity Hardware](https://lirias.kuleuven.be/bitstream/123456789/473761/1/acsac2014.pdf) presented at ACSAC 2014. *If you use these tools in your research, please reference this paper.* Most code is open source, and contributions are welcome. The code of the constant jammer can be requested but is not available publicly. Don't worry, we won't bite.

- April 2016: we now support Linux kernels 3.0 up to and including 4.4! See the [modwifi-4.4-1.tar.gz](releases/modwifi-4.4-1.tar.gz) release! This has been tested on Arch Linux and Ubuntu 15.10.
- September 2019: we have a release that **supports kernels 5.3 and below**. See the [modwifi-5.3-rc4-1.tar.gz](releases/modwifi-5.3-rc4-1.tar.gz) release. This was tested on kernel 4.9.0-9 and 5.2.9.

## Table of Contents

* [Quick Start](#quick-start)
* [Basic Usage](#basic-usage)
    * [Reactive Jamming](#reactive-jamming)
    * [Disabling Carrier Sense](#disabling-carrier-sense)
    * [Constant Jamming](#constant-jamming)
    * [Unfair Channel Usage](#unfair-channel-usage)
    * [Forcing Corrupt Packets](#forcing-corrupt-packets)
    * [Channel MitM and TKIP Broadcast Attack](#channel-mitm-and-tkip-broadcast-attack)
* [Troubleshooting](#troubleshooting)
* [Installation and Source Code](#installation-and-source-code)
* [Raspberry Pi Support](#raspberry-pi-support)
* [Repositories](#repositories)
* [Supporting new kernels](#supporting-new-kernels)

## Quick Start

You can [download a VMWare image](http://people.cs.kuleuven.be/~mathy.vanhoef/modwifi/Xubuntu-Modwifi.7z) that has the drivers, firmware, and user-land tools preinstalled. Just boot it, plug-in the USB dongle, and start experimenting! **The password of the account modwifi is modwifi**. Once booted, you can execute (the public) attacks below.

## Basic Usage

This section describes the attacks that can be executed. We assumed you already downloaded the VMWare image or manually installed the drivers and firmware (see the section "Installation" to install drivers on your existing machine).

*Before doing any attacks it is recommended to disable WiFi.* In particular I mean disabling WiFi in your network manager. Most graphical network managers have an option somewhere named "Enable Wi-Fi". Make sure it's not selected. If you can't find it, perhaps you can disable in the terminal with `sudo nmcli nm wifi off`. Once you have disabled WiFi your OS won't interfere with our attacks.

*If RF-kill is enabled* we'll have to turn it off. Some distributions set RF-kill on after disabling WiFi. But we still want to actually use our WiFi devices. So execute:

```bash
sudo apt-get install rfkill
sudo rfkill unblock wifi
```

#### Reactive Jamming

Our current implementation of our reactive jammer allows you to block an Access Point. More precisely, all beacons and probe responses will be jammed. Execute it using:

```bash
modwifi@ubuntu:~/modwifi/tools$ sudo rfkill unblock wifi
modwifi@ubuntu:~/modwifi/tools$ sudo iw wlan0 set type monitor
modwifi@ubuntu:~/modwifi/tools$ sudo ifconfig wlan0 up
modwifi@ubuntu:~/modwifi/tools$ sudo iw wlan0 set channel 11
modwifi@ubuntu:~/modwifi/tools$ sudo ./reactivejam -i wlan0 -s "Home Network"
```

**The first three commands need to be executed only once** after plugging in your dongle. To get the interface name of the wireless card you can execute `iwconfig`. In this case our targeted AP was on channel 11, but remember that your targeted AP may be on a different channel.

You can stop the reactive jammer using CTRL+C. It may take a few seconds before it actually stops. By modifying the firmware you can reactive jam any kind of packets you like. For example, you could jam all packets of a specific client. Note that only medium to large packets can be reliably jammed (see our paper).

You can verify that this works by monitoring the channel with a second device. Make sure that this device also reports corrupted frames using:

```bash
sudo iw wlan1 set monitor fcsfail
```

This will instruct the driver to also pass corrupted frames to the userland (when in monitor mode). Be warned though, not all drivers properly support this flag. Some will always show corrupted frames. Others will never show corrupted frames. Our drivers and firmware handle this flag correctly!

#### Disabling Carrier Sense

Want to disable carrier sense in order to perform an experiment? Then execute this:

```bash
modwifi@ubuntu:~$ sudo su
root@ubuntu:~$ mount -t debugfs none /sys/kernel/debug
root@ubuntu:~$ cd /sys/kernel/debug/ieee80211/phy*/ath9k_htc/registers/
root@ubuntu:~$ echo 1 > force_channel_idle
root@ubuntu:~$ echo 1 > ignore_virt_cs
```

Writing 1 to `force_channel_idle` disables physical carrier sense (channel is busy). Writing 1 to `ignore_virt_cs` disables virtual carrier sense (RTS/CTS). Random backoff parameters can also be changed.

#### Constant Jamming

If you have the firmware capable of doing constant jamming, you can execute:

```bash
modwifi@ubuntu:~/modwifi/tools$ sudo iw wlan0 set type monitor
modwifi@ubuntu:~/modwifi/tools$ sudo ifconfig wlan0 up
modwifi@ubuntu:~/modwifi/tools$ sudo ./constantjam wlan0 6
```

This performs constant jamming on channel 6. Because channels overlap, nearby channels will also be jammed. Remember that the constant jamming implementation is not public, but can be requested privately.

#### Unfair Channel Usage

The specific scripts we used to easily configure a device to act unfairly are not public. The reason behind this is that it's hard to defend against these kind of attacks. However, some parameters can still be accessed as `debugfs` entries in `/sys/kernel/debug/ieee80211/phy*/ath9k_htc/registers/`.

#### Forcing Corrupt Packets

You can force the wireless chip to calculate a wrong CRC (FCS) using:

```bash
modwifi@ubuntu:~$ sudo su
root@ubuntu:~$ mount -t debugfs none /sys/kernel/debug
root@ubuntu:~$ cd /sys/kernel/debug/ieee80211/phy*/ath9k_htc/registers/
root@ubuntu:~$ echo 1 > diag_corrupt_fcs
```

#### Channel MitM and TKIP Broadcast Attack

This is an advanced attack and not for the fainthearted. It clones an existing Access Point on a different channel. This allows us to reliably manipulate encrypted traffic. We used this to break TKIP. See [our paper]() for details. An example on how we used it to verify that our awesome-sauce attacks work:

```bash
modwifi@ubuntu:~/modwifi/tools$ sudo ./channelmitm -a wlan4 -c wlan5 -j wlan3 -s testnetwork -d mitm.pcap --dual
```

## Troubleshooting

If an attack or device is not working, you can try the following steps to get it working again:

1. Change the channel of the device. This will reset the wireless chip in the dongle, and perhaps fix the issue.
2. Bring the device up and down using `ifconfig` or `ip link`. This should reset even more settings than just changing the channel.
3. Unplug the device and plug it back it. This reloads the complete firmware.
4. If all else fails, reboot your computer.

If you can reproduce a bug, feel free to file a bug report.

Another few remarks when using our tools, and doing wireless hacking in general:

- You can only change the channel of a monitor device when no other (virtual) interface is active. So if you have a `monX` interface, you need to bring down (`ifconfig wlanX down`) all other interfaces (which use that device) first.
- In general you want to kill other processes that are trying to use/configure your WiFi device. Tools like [`airmon-zc`](http://svn.aircrack-ng.org/tags/1.2-beta3/manpages/airmon-zc.8) can help detect which processes might be interfering. Note that `airmon-zc` is the successor of the older `airmon-ng` tool.

## Installation and Source Code

You can also install the latest drivers and firmware on your own machine. The quickest method is to grab [one of our release packages](https://github.com/vanhoefm/modwifi/raw/master/releases/). Only your wireless stack and drivers will be replaced, all other drivers will remain the same (if you use other wifi devices as well, compile them too). Normal usage of WiFi still works perfectly when these drivers are installed (I use these drivers myself :).

The installation instructions are:

```bash
mkdir modwifi && cd modwifi
wget https://github.com/vanhoefm/modwifi/raw/master/releases/modwifi-4.4-1.tar.gz
tar -xf modwifi-4.4-1.tar.gz

apt-get install build-essential libncurses-dev bison flex libssl-dev
cd drivers && make defconfig-ath9k-debug
make
sudo make install
cd ..

# Note: the location and name of firmware files on your machine may be different
compgen -G /lib/firmware/ath9k_htc/*backup || for FILE in /lib/firmware/ath9k_htc/*; do sudo cp $FILE ${FILE}_backup; done
sudo cp target_firmware/htc_7010.fw /lib/firmware/ath9k_htc/htc_7010-1.4.0.fw
sudo cp target_firmware/htc_9271.fw /lib/firmware/ath9k_htc/htc_9271-1.4.0.fw

sudo apt-get install g++ libssl-dev libnl-3-dev libnl-genl-3-dev cmake
cd tools && cmake CMakeLists.txt && make all
```

**Reboot** so our new drivers will be used. After that you should be good to go. That is, plug in your dongle, and execute the compiled tools.

Note that this only compiles and installs the ath9k drivers. If you want to use modwifi, and at the same time control other wireless networks cards on the kernel, modify and use the appropriate `defconfig-*` file (e.g. include the appropriate flags in `defconfig-ath9k-debug` so the drivers you need are also compiled).

If you want to compile the firmware as well, clone the [ath9k-htc repository](https://github.com/vanhoefm/modwifi-ath9k-htc), and follow the instructions there. If you want to modify the driver, you can modify the downloaded code in `modwifi-YYYYMMDD.tar.gz`. You can put that code in your own repository to keep track of changes, and send us patches based on this. Alternatively, the more correct but also significantly more tedious method, would be to clone the research branch of [our forked Linux kernel](https://github.com/vanhoefm/modwifi-linux). The driver can be extracted from the kernel code using the [backports](https://backports.wiki.kernel.org/index.php/Main_Page) project. You can then install the drivers only (so without modifying your own kernel).

## Raspberry Pi Support

Our drivers and firmware can be run on a Raspberry Pi. We tested this using raspbian. In order to get it working first download and update some dependencies:

```bash
sudo apt-get install linux-image-3.12-1-rpi linux-headers-3.12-1 g++-4.7 iw
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.7 50
```

As you can see, we tested this on the 3.12-1-rpi kernel. You can use another kernel if you want, just be sure to download the kernel headers. To enable the 3.12-1-rpi kernel we just downloaded edit `/boot/config.txt` and append:

	kernel=vmlinuz-3.12-1-rpi
	initramfs initrd.img-3.12-1-rpi followkernel

And to assure our raspberry pi will recognize the device when we plug it in, execute:

	echo "ath9k_htc" | sudo tee -a /etc/modules

Everything is now ready to install our drivers and firmware. Just **follow the instructions under section "Installation"**. Compilation of the drivers can take a while. Finally we have to prevent raspbian from automatically trying to enable and manage WiFi (this interferes with our attacks). First edit `/etc/network/interfaces` and comment out the following two lines:

	#allow-hotplug wlan0
	#iface wlan0 inet manual

Now edit `/etc/default/ifplugd` and change the `INTERFACES` and `HOTPLUG_INTERFACES` to:

	INTERFACES="eth0"
	HOTPLUG_INTERFACES="eth0"
	ARGS="-q -f -u0 -d10 -w -I"
	SUSPEND_ACTION="stop"

This will prevent raspbian from automatically enabling and managing the wireless interface (so we can first put the device in monitor mode and only then enable it). You can now compile the tools and execute the attacks!

## Repositories

The work is divided over several git repositories:

1. **Linux:** [Forked Linux kernel](https://github.com/vanhoefm/modwifi-linux) to make driver modifications.
2. **Backports:** [Fork of the backports](https://github.com/vanhoefm/modwifi-backports) projects so we can backport our drivers to older kernels.
3. **Ath9k-htc**: [Forked firmware code](https://github.com/vanhoefm/modwifi-ath9k-htc) to implement the core of our attacks.
4. **Tools:** New repository for our [user-land tools](https://github.com/vanhoefm/modwifi-tools).

You can download all repositories at once using the following commands:

```bash
mkdir modwifi && cd modwifi
bash <(curl -s https://raw.githubusercontent.com/vanhoefm/modwifi/master/init.sh)
```

To compile the Linux and ath9k-htc firmware, read the documentation of these projects. To backport the modified drivers using the backports project, also see the official documentation of that project. Finally, our tools can be compiled using a simple `make all`. Apart from the `tools` repositories, all work and modifications are performed on the `research` branch. When a new Linux kernel (or firmware) is released, we can easily merge with it. As a result **our code is relatively easy to keep up-to-date**.

For those who also want to start hacking away at the driver and firmware, I recommend first reviewing our patches. This allows you to study what our changes do, and inspect the firmware code at small chunks one at a time. That way it's easier to learn step by step. Maybe you will even find bugs or can make improvements (let us know). Also, in the `ath9k-htc` repository, there is a directory called `docs`. While still terse to read, these documents should be an excellent guide while reading and understanding the code.

If you have any questions, don't hesitate to send us a mail.

## Supporting new kernels

We rely on the [driver backports project](https://backports.wiki.kernel.org/index.php/Main_Page) to ship our modified drivers (as installable modules) to older kernels. This provides two advantages: (1) the drivers can be installed as modules, meaning user don't have to (re)compile a linux kernel; and (2) these drivers (i.e., modules) are compatible with many recent kernel versions. Officially the [backports project tracks the linux-next tree](https://backports.wiki.kernel.org/index.php/Documentation/backports/hacking#Git_trees_you_will_need). This means it extracts, and backports, recents drivers (as loadable modules) from the linux-next tree. However, we base our code on [Linus' tree](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git), because [basing code on linux-next is not really possible](https://lwn.net/Articles/289013/).

To target a new kernel there are two cases:

1. There is a `linux-x.y.z` branch of the kernel you want to target. Checkout this branch, use it to extract modified drivers. Should be relatively simple.
2. We need to create our own `linux-x.y.z` branch. I call my own batches of this type `mathy-x.y.z`. It is best to first get backports working against a linux-next tag that was tested by the backports project itself. Then I found that, with some minor patches, backports will also work against Linus' tree of a specific version tag that is close to the tested linux-next snapshot. So find the latest linux-next snapshot that is compatible with the backports project. [How to get linux-next](https://www.kernel.org/doc/man-pages/linux-next.html). You may need to use [linux-next-history](http://git.kernel.org/cgit/linux/kernel/git/next/linux-next-history.git) for this, which [contains all linux-next tags](http://lkml.iu.edu/hypermail/linux/kernel/1108.0/00555.html) ever created. It may be that backports cannot cleanly extract the driver code. If so, crease a new research branch in the backports repository, and modify the patches so everything cleanly applies and compiles.

**Use python2 when using backports**. Some scripts may fail if python3 is the default.

Note that branches named `mathy-x.y.z` are custom branches, I personally use them to create backports for my _currently running_ linux kernel. For example, `mathy-4.7.y` can take code from linux-next, and will compile on Linux 4.7.4. However, it may not compile on older kernels! While working on backports, you may find it useful to use `rediff` from the `patchutils` package to manually change patch files.

