<!-- vim: set spell spelllang=en_us: -->

# U-Boot

Website: [https://www.denx.de/wiki/U-Boot](https://www.denx.de/wiki/U-Boot)

U-Boot ("Das U-Boot") is a boot loader for embedded systems. It could
theoretically be used for your desktop machine as well, but it is mainly
intended to be used in embedded systems. U-Boot is configured very similarly to
the Linux kernel: there is a .config file which can be generated and edited with
an ncurses-based tool.

There's not a whole lot to say here about U-Boot. For the most part, you'll be
modifying the source for your chip to add your own board's definitions and such
to it, assuming the defaults for that don't already work. This is heavily
dependent on what you need to do with your system and what hardware you have
chosen, so I can't say much about that here.

U-Boot can boot from the network, various types of flash, hard drives, and so
on. Just about any type of device is supported, or can be supported if you add
the code for it. It can also be configured to use a serial console, and includes
a bunch of handy command-line tools in a POSIX sh-like shell. These tools
include (but are not limited to):

- `mw` for writing memory of various sizes at arbitrary addresses
- `md` for dumping memory at arbitrary addresses
- commands for booting the system manually

Effectively, you can manually manipulate the majority of your system from the
U-Boot shell. This allows you to test hardware without a device tree, kernel, or
anything getting in your way--you can manually modify whichever registers you
need.

There isn't much more to say about U-Boot here, as you will be doing a lot that
is extremely specific to your use case. Look for docs on that, or roll your own.
