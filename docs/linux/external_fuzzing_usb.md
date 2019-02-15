External USB fuzzing for Linux kernel
=====================================

https://www.offensivecon.org/speakers/2019/andrey-konovalov.html

# USB fuzzing with syzkaller

This page describes the current state of external USB fuzzing support in syzkaller.

It's still in development and things might change, in particular two things need to be done: 1. land syzkaller changes ([pull request](https://github.com/google/syzkaller/pull/982)) and 2. land the kernel changes (see the patches in the instructions below). See the OffensiveCon [slides](https://docs.google.com/presentation/d/1z-giB9kom17Lk21YEjmceiNUVYeI6yIaG5_gZ3vKC-M/edit?usp=sharing) for more details.

This allowed to find over [80 bugs](/docs/linux/found_bugs_usb.md) in the Linux kernel USB stack so far.

How to set this up:

1. Checkout upstream Linux kernel (all the patches linked below are based on 5.0-rc5).

2. Apply three patches to the kernel: [1](/tools/usb/0001-kcov-remote-coverage-support.patch), [2](/tools/usb/0002-usb-kcov-annotate-hub_event.patch) and [3](/tools/usb/0003-usb-fuzzer-main-usb-gadget-fuzzer-driver.patch). Patches 1 and 2 make it possible to collect coverage from the USB subsystem. Patch 3 implements a GadgetFS-like interface for emulating USB devices from userspace.

3. Configure and build the kernel. You need to enable `CONFIG_USB_FUZZER=y` and `CONFIG_USB_DUMMY_HCD=y`:

   ```
   menu config -> Device Drivers -> USB Support ->
     -> USB Gadget Support (enable) -> 
       -> USB Peripheral Controller -> Dummy HCD (enable)
       -> USB Gadget Fuzzer (enable)
   ```

   [This](/tools/usb/kernel.config) is the config I used for testing the upstream kernel.

4. Optionally update syzkaller descriptions by extracting USB device info by using the instructions below.

5. Checkout syzkaller `usb-fuzzer` branch, build syzkaller as usual.

6. Enable `syz_usb_connect`, `syz_usb_disconnect`, `syz_usb_control_io` and `syz_usb_ep_write` syscalls in the manager config.

7. Set `sandbox` to `none` in the manager config.

8. Run.

Syzkaller descriptions for USB fuzzing can be found here: [1](/sys/linux/vusb.txt) and [2](/sys/linux/vusb_ids.txt).


## Running reproducers with Raspberry Pi Zero W

1. Download `raspbian-stretch-lite.img` from [here](https://www.raspberrypi.org/downloads/raspbian/).

2. Flash the image into an SD card as described [here](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

3. Enable UART as described [here](https://www.raspberrypi.org/documentation/configuration/uart.md).

4. Boot the board and get a shell over UART as described [here](https://learn.adafruit.com/raspberry-pi-zero-creation/give-it-life). You'll need a USB-UART module for that. The default login credentials are `pi` and `raspberry`.

5. Get the board connected to the internet (plug in a USB Ethernet adapter or follow [this](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)).

6. Update: `sudo apt-get update && sudo apt-get dist-upgrade && sudo rpi-update && sudo reboot`.

7. Install useful packages: `sudo apt-get install vim git`.

8. Download and install Go:

``` bash
curl https://dl.google.com/go/go1.10.8.linux-armv6l.tar.gz -o go1.10.8.linux-armv6l.tar.gz
tar -xf go1.10.8.linux-armv6l.tar.gz
mv go goroot-1.10.8
mkdir gopath-1.10.8
export GOPATH=~/gopath-1.10.8
export GOROOT=~/goroot-1.10.8
export PATH=~/goroot-1.10.8/bin:$PATH
export PATH=~/gopath-1.10.8/bin:$PATH
```

9. Download syzkaller, apply [this patch](https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/syzkaller.patch) and build `syz-executor`:

``` bash
go get -u -d github.com/google/syzkaller/...
cd ~/gopath-1.10.8/src/github.com/google/syzkaller
git checkout usb-fuzzer
wget https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/syzkaller.patch
git apply ./syzkaller.patch
make executor
mkdir ~/syz-bin
cp bin/linux_arm/syz-executor ~/syz-bin/
```

10. Build `syz-exeprog` on your host machine usign the `usb-fuzzer` branch for arm32 with `make TARGETARCH=arm execprog` and copy to `~/syz-bin` onto the SD card.

11. Make sure that ou can now execute syzkaller programs:

``` bash
cat socket.log
r0 = socket$inet_tcp(0x2, 0x1, 0x0)
sudo ./syz-bin/syz-execprog -executor ./syz-bin/syz-executor -threaded=0 -collide=0 -procs=1 -nocgroups -nonetdev -nonetreset -notun -debug socket.log
```

12. Setup the dwc2 USB gadget driver:

```
echo "dtoverlay=dwc2" | sudo tee -a /boot/config.txt
echo "dwc2" | sudo tee -a /etc/modules
sudo reboot
```

13. Get Linux kernel headers following [this](https://github.com/notro/rpi-source/wiki).

14. Download and build the fuzzer module:

``` bash
mkdir module
cd module
wget https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/Makefile
wget https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/fuzzer.c
wget https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/fuzzer.h
make
```

15. Insert the module with `insmod fuzzer.ko`.

16. Build and test the keyboard emulator:

``` bash
# Connect the board to some USB host.
wget https://raw.githubusercontent.com/google/syzkaller/usb-fuzzer/tools/usb/keyboard.c
gcc keyboard.c -o keyboard
sudo ./keyboard
# Make sure you see the letter 'x' being entered on the host.
```

17. You should now be able to execute syzkaller USB programs:

```
cat usb.log
syz_usb_connect(0x2, 0x36, &(0x7f0000000000)={0x12, 0x1, 0x0, 0x0, 0x0, 0x0, 0x40, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x1, [{0x9, 0x2, 0x24, 0x0, 0x0, 0x0, 0x0, 0x0, [{0x9, 0x4, 0x0, 0x0, 0x2, 0x0, 0x0, 0x0, 0x7, [], [{0x9, 0x5, 0x78f, 0x2}, {0x9, 0x5, 0x6, 0x2}]}]}]}, &(0x7f0000233000)=@id_7202={0x1286, 0x2046, 0x2da3, 0x6, 0x4, 0x8, 0xff, 0x4, 0x1, 0x465d}, 0x0)
sudo ./syz-bin/syz-execprog -executor ./syz-bin/syz-executor -threaded=0 -collide=0 -procs=1 -nocgroups -nonetdev -nonetreset -notun -debug usb.log
```

18. Follow [this](https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md) to setup Wi-Fi hotspot.

19. Follow [this](https://www.raspberrypi.org/documentation/remote-access/ssh/) to enable ssh.

20. Optionally solder [Zero Stem](https://zerostem.io/) onto your Raspberry Pi Zero W.

21. You can now connect the board to an arbitrary USB port, wait for it to boot, join its Wi-Fi network, ssh onto it, and run arbitrary syzkaller USB programs.
