<!-- vim: set spell spelllang=en_us: -->

# Device Trees

Device trees are a basic file format used to describe hardware devices and the
relationships between them in a very generic way. The current standard is
available at [devicetree.org](https://www.devicetree.org). The standard is
maintained by a community associated with, among others, ARM and Linaro.

While the standard for the device tree is available from devicetree.org, this is
only a very generic and general specification. The kernel sources include
documentation for many devices supported and how to specify them in the device
tree. You can find them in the kernel source tree at
`Documentation/devicetree/bindings`.

**Note:** the Linux kernel build process involves running the C preprocessor
(cpp) on the device tree sources. This is done so that `#include` and
preprocessor macros can be used to simplify the contents of your device tree and
make them more readable and consistent. For a list of files you can include in
your device tree, check `include/dt-bindings`. For example, for common GPIO
definitions, you could put `#include <dt-bindings/gpio/gpio.h>` at the top of
your device tree. This is also used to include of device tree fragments, meaning
you can split your device tree into multiple smaller pieces or extend a larger
one (such as the base one for your chip) with your own bindings.

Device trees for ARM devices are located in `arch/arm/boot/dts`.

<!-- vim-markdown-toc GFM -->

* [Format](#format)
* [Device Types](#device-types)
* [Pin Mapping](#pin-mapping)
* [Device Mapping](#device-mapping)
* [References](#references)

<!-- vim-markdown-toc -->

## Format

For the full details on the format of device trees, see the standard linked
above.

## Device Types

*If you are in doubt, see the Linux kernel documentation on device trees in
`Documentation/devicetree/bindings`.*

Linux supports a lot of different classes of device. These correlate to the
various subsystems in the kernel which handle them. Hardware monitoring (hwmon)
devices are used for things like fan controllers, thermal controllers,
regulators, and so on. The industrial I/O (iio) subsystem is used for things
like ADCs and DACs. You can add CAN controllers/transceivers, net devices such
as Ethernet controllers, serial UART/RS-484 devices, GPIO port expanders, FPGAs,
LEDs, and many, many more things to your device tree. These are categories that
are *not* related to any particular hardware interface--you may have a driver
for a GPIO port expander attached to SPI, I2C, or some other bus; you might have
an LED connected to a GPIO, a PWM output, or an LED controller available over
SPI or I2C. Your fan controller may be an I2C device, SPI device, or whatever
else. This is where the driver abstracts away the hardware for you. To the
kernel and user-space, there's just a fan controller. You and the driver both
know that it's attached at address `0x1a` over I2C, but your software need not
deal with I2C, bus arbitration between different devices, synchronizing access
to the bus between drivers, or setting registers. Instead, you will access the
files in `/sys/class/hwmon/hwmonX/` (where X is the enumerator the kernel has
assigned to that particular device) and read/write values there at a high level.

## Pin Mapping

This all sounds very convenient--because it is! The downside is that you will
likely spend a decent amount of time fiddling with the device tree, finding the
right way to use a device, and setting up pins properly. This can be annoying.

*Note: you will likely want to just modify an existing device tree for your
chip, as there likely is one. If not, you'll need to read up on what is required
to make your board functional.*

Let's add an I2C bus. For this example, I will be writing part of a device tree
(not a fully-functional one) for an AM3358-based chip. First, I will start by
including the files that TI has already written for me:

```c
#include "am33xx.dtsi"
```

Next, I will define which pins are used for the I2C bus. Note that in the
file included above (and the others it includes), `i2c0` was already defined.
This means that instead of referring to it with `i2c0`, I will refer to it with
`&i2c0`, which basically appends properties and child nodes to the `i2c0` node
rather than defining a new one.

```
&i2c0 {
  pinctrl-names = "default";
  pinctrl-0 = <&i2c0_pins>;

  status = "okay";
  clock-frequency = <400000>; /* 400kHz */
};
```

I have specified that the pin control node used for this device is `i2c0_pins`,
which will be a node that appears within the device's whole pin mapping. For
details on the `pinctrl-names` and `pinctrl-0` values, see
`Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt`.
`status` is a common property in device trees which can be used to enable or
disable a device. `clock-frequency` (among others) is an option specific to I2C
controllers.

Since am33xx.dtsi has already declared that `am33xx_pinmux` is the node defining
our pin mapping and that it is compatible with `pinctrl-single`, I will use that
to define my pins. Let's start by doing this the ugly way. TI has already
defined what the base address for pin configuration is: `0x44e10800`. However,
this is actually already offset by `0x800` from the base pad configuration
register, so when we read the address for a pad configuration register in the
manual, we need to subtract `0x800`. Thus, in order to map the `I2C0_SDA` and
`I2C0_SCL` pins to actually be in those modes (that happens to be mode 0 for
those particular pins), I need to get their addresses (`0x998` and `0x98c`) and
subtract `0x800`. This is getting complicated. Note that they have also already
defined that I need to have 3 values per pin: offset, configuration, and mux
mode. Thus, to configure the pins as inputs with pull-ups enabled in mode 0,
I will read the manual, figure out which bits need to be set in the
configuration register, and put this in there:

```
&am33xx_pinmux {
  i2c0_pins: pinmux-i2c0-pins {
    pinctrl-single,pins = <
      0x198 0x108 0   /* I2C0_SDA.I2C0_SDA */
      0x18c 0x108 0   /* I2C0_SCL.I2C0_SCL */
    >;
  };
};
```

This is pretty awful. It's terrible to maintain, it's hard to read, and it is
not very clear what is going on here. However, this can be made significantly
better. Device trees support basic math, which means we can make this a bit
clearer:

```
&am33xx_pinmux {
  i2c0_pins: pinmux-i2c0-pins {
    pinctrl-single,pins = <
      (0x998 - 0x800)  ((1 << 8) | (1 << 3))  0   /* I2C0_SDA.I2C0_SDA */
      (0x98c - 0x800)  ((1 << 8) | (1 << 3))  0   /* I2C0_SCL.I2C0_SCL */
    >;
  };
};
```

That's a little clearer. However, remember that the build process includes using
the C preprocessor, which means we can use macros. Conveniently, TI has already
created a bunch of those for us in `include/dt-bindings/pinctrl/omap.h`, which
is included by the file we included earlier. We can now simplify this much more
using those macros (you could define your own if some don't already exist):

```
&am33xx_pinmux {
  i2c0_pins: pinmux-i2c0-pins {
    pinctrl-single,pins = <
      AM33XX_IOPAD(0x998, PIN_INPUT_PULLUP | MUX_MODE0) /* I2C0_SDA.I2C0_SDA */
      AM33XX_IOPAD(0x98c, PIN_INPUT_PULLUP | MUX_MODE0) /* I2C0_SCL.I2C0_SCL */
    >;
  };
};
```

In fact, TI has even added another header which includes a bunch of definitions
for the names of the pad control registers, meaning that in this case we don't
even have to use the pin number. Instead, we can use another macro plus the
macros that define the pad names. This would make things even easier.

## Device Mapping

Following the previous example, we will want to add an I2C slave device. We can
add a Maxim MAX6697 temperature monitor. This will show up as a hwmon device in
Linux. For details on the options available for this device, see
`Documentation/devicetree/bindings/hwmon/max6697.txt`. Let's go ahead and add to
the `i2c0` node like we did before. I'll add this inside the one I wrote before
so that I don't have multiple `&i2c0` nodes, which just gets hard to maintain.
We will specify that our device is at address `0x1a` like the example in the
documentation:

```
&i2c0 {
  pinctrl-names = "default";
  pinctrl-0 = <&i2c0_pins>;

  status = "okay";
  clock-frequency = <400000>; /* 400kHz */

  /*
    temp-sensor is an arbitrary name, but the names sometimes mean something
    and change how they are handled by the kernel, so make sure to read the
    documentation for the type of device you're adding!
  */

  temp-sensor@1a {
    /* properties common to all I2C devices */
    compatible = "maxim,max6697";
    reg = <0x1a>;

    /* device-specific properties */
    smbus-timeout-disable;
    resistance-cancellation;
    alert-mask = <0x72>;
    over-temperature-mask = <0x7f>;
  };
};
```

Here we added our temperature monitor at address `0x1a` and specified that it is
compatible with the driver `maxim,max6697`. Those properties (`compatible` and
`reg`) are common to all I2C devices--they're also common with all devices, in
fact. The rest of the properties are specific properties that the driver will
understand. The ones without values are booleans: if they are specified they are
true, otherwise they are false. See the documentation for what the options
available for your device are.

This is really not all that difficult, and there are similar idioms for other
things: a SPI bus will have a list of GPIOs or other pins used as chip selects
and the devices will be specified as offsets of those, rather than having
addresses like I2C. For example, if your SPI slave is at chip select 0 (the
first one specified for that controller), you would have `@0` at the end of the
child device node's name and `reg = <0>;` inside the child node.

Now that we have added our hwmon device, and assuming our kernel has the driver
for the device built in (you will find it in the device drivers section of the
kernel configuration menu), we can now find our device at
`/sys/class/hwmon/hwmon0` (or some other number, but assuming there are no other
hwmon devices, it will be 0). We can see how the device works in
`Documentation/hwmon/max6697.rst`. By writing and reading those files, we can
read the temperatures in our system, poll for alarms, and change alarm
thresholds--all without tediously writing I2C frames, setting up registers, and
so on.

----

## References

1. [Device tree usage model (Linux kernel documentation)
](https://www.kernel.org/doc/Documentation/devicetree/usage-model.txt)
