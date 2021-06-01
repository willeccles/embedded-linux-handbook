<!-- vim: set spell spelllang=en_us: -->

# Basic Concepts of Embedded Linux

The architecture of your embedded system will likely look a lot like this:

```
+-----------------------------------+
| user-space code                   |
+-----------------------------------+
| kernel-space code (drivers, etc.) |
+-----------------------------------+
| device tree                       |
+-----------------------------------+
| boot loader                       |
+-----------------------------------+
```

Your user-space program(s) will communicate with kernel-space drivers to access
hardware devices, manage scheduling, and so on. The kernel will get the details
of which devices are installed in the system and where they are from the device
tree and will then pass that information on to the various device drivers.

The boot loader is in charge of preparing the system, enabling clocks, powering
up peripherals, and so on. It will then load the Linux kernel and execute it
with some set of arguments that you can choose.

<!-- vim-markdown-toc GFM -->

* [Boot Loader](#boot-loader)
* [Device Tree](#device-tree)
* [Kernel Drivers](#kernel-drivers)
* [User-space](#user-space)

<!-- vim-markdown-toc -->

## Boot Loader

In the boot loader, you will initialize the very basics of the processor. This
includes powering it on, enabling any peripherals required for boot (such as
a serial console for boot loader/kernel output, ethernet for network boot, or
writing identifying information to an EEPROM), and then actually booting the
kernel from the network, an MMC device, or a some other type of storage device.

For many systems, little will need to be done in the boot loader. For the most
part, cloning the u-boot sources, doing a little bit of configuring and
patching, and then building it is enough to have a working bootloader. YMMV.

## Device Tree

The device tree is arguably the most important (and worst-documented) part of
the entire process. Some vendors have or had proprietary file formats for
configuring hardware devices for the kernel. Many opted to maintain a fork of
the kernel for only marginally-different ARM devices, which resulted in a lot of
fragmented information and device support. As a result, Linux has settled on
using the device tree format for different boards. In order to make your system
work, you will create a device tree for your board, likely based on a reference
one for an EVM or developer kit for your chosen processor.

The device tree will tell the kernel which clocks are where, what registers are
used for which devices, what kinds of devices you have, and which drivers should
be used with them. For example, say you have an I2C bus with two slave devices,
`mydeviceA` and `mydeviceB`. They will be supported by `mydriverX` and
`mydriverY`. Their address are `0x20` and `0x21`. You might have something like
this in your device tree:

```
/* i2c0 will have been setup elsewhere, we will just add devices to it */
&i2c0 {
  mydeviceA@20 {
    compatible = "mydriverX";
    reg = <0x20>;
    /* each device may have driver-specific properties or other common i2c
    slave properties */
  };

  mydeviceB@21 {
    compatible = "mydriverY";
    reg = <0x21>;
  };
};
```

Linux will understand that it should load the two drivers and tell them about
the devices at addresses `0x20` and `0x21` on the bus referred to as `i2c0`,
which was setup elsewhere. This way, you will be able to simply talk to the
device driver at a high level while the critical timing and IO happens in
kernel-space without context-switching. This will be more efficient than if you
use the user-space I2C driver to communicate with the device directly, and it's
also much easier to maintain your userspace code.

If possible, you should write a kernel device driver for your custom devices (or
those that don't already have drivers in the kernel source tree). If you can't
or don't want to, then you can just fall back to using user-space I/O drivers
for I2C, SPI, GPIO, etc.

## Kernel Drivers

Kernel drivers will expose a device somewhere in the sysfs. Read the
documentation for your device's driver (usually included with the kernel source,
but some companies did not feel the need to write any documentation--looking at
you, MAX31790) to learn what the driver exposes in the sysfs for your devices.
These things will appear as files, which you can open and read and write, but
they will only accept certain inputs and will give you certain outputs. Some
files are made for polling rather than reading/writing, and so on. It all
depends on the device.

## User-space

Your user-space program is where you will write the main control logic for your
program unless you're a masochist and prefer to write your entire program in the
kernel. This doesn't work very well for a lot of things and you shouldn't do it.
Your program will be run probably as root (or by another user with sufficient
permissions to access the required devices) and will communicate with the device
drivers in the kernel to control the system and acquire input. This is the easy
part; it's just like any other program, though you may need to deal with
real-time scheduling and such.
