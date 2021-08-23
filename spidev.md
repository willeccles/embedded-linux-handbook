<!-- vim: set spell spelllang=en_us: -->

# Using SPI From Userspace Applications (spidev)

While the majority of hardware interfacing should be done in kernel-space (using
device driver kernel modules), some applications may be simplified by doing I/O
from userspace. When using SPI from userspace, the spidev driver is used for
generic SPI devices.

## Table of Contents
<!-- vim-markdown-toc GFM -->

* [Initializing a SPI Device](#initializing-a-spi-device)
* [Using Spidev for TX/RX](#using-spidev-for-txrx)
* [Advanced Userspace SPI Features](#advanced-userspace-spi-features)
* [References](#references)

<!-- vim-markdown-toc -->

## Initializing a SPI Device

While the kernel does have some documentation<sup>1</sup> on using spidev, some
of the best information available can be gleaned from the associated headers and
kernel source, which is a pretty common theme with Linux drivers and interfaces
(unfortunately). When you are looking for the various available options for
things like the SPI mode and device configuration, you should check in the
kernel source tree in include/uapi/linux/spi/spidev.h or in the installed header
at /usr/include/linux/spi/spidev.h (or a similar location for cross-compile
toolchains and such). Additionally, the kernel source includes an example
program using the SPI userspace API, which is a good reference.

Here is an example of opening a SPI device and configuring it for 10MHz in SPI
mode 2 with 8 bits per word:

```c
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <fcntl.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <unistd.h>

#include <linux/spi/spidev.h>
#include <linux/types.h>

int openspidev(int bus, int cs) {
  char devpath[35];

  if (bus < 0 || cs < 0) {
    errno = EINVAL;
    return -1;
  }

  snprintf(devpath, 35, "/dev/spidev%d.%d", bus, cs);

  // check if the device exists
  if (access(devpath, F_OK) != 0) {
    return -1;
  }

  return open(devpath, O_SYNC | O_RDWR | O_CLOEXEC);
}

int main(int argc, char** argv) {
  int bus;
  int cs;
  int fd;
  uint8_t tmp8;
  uint32_t tmp32;
  int ret;

  if (argc != 3) {
    fprintf(stderr, "usage: %s bus cs\n", argv[0]);
    return 1;
  }

  bus = atoi(argv[1]);
  cs = atoi(argv[2]);

  fd = openspidev(bus, cs);
  if (fd < 0) {
    perror("openspidev");
    return 1;
  }

  /* configure the SPI device */

  // bits per word = 8
  tmp8 = 8;
  ret = ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &tmp8);
  if (ret < 0) {
    perror("ioctl: SPI_IOC_WR_BITS_PER_WORD");
    return 1;
  }

  // max speed = 1MHz
  tmp32 = 1 * 1000 * 1000;
  ret = ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &tmp32);
  if (ret < 0) {
    perror("ioctl: SPI_IOC_WR_MAX_SPEED_HZ");
    return 1;
  }

  // set mode to mode 2
  // you can use the additional mode flags which don't fit in a single byte
  // here, since we will use WR_MODE32, while the normal WR_MODE only accepts
  // a uin8_t
  tmp32 = SPI_MODE_2; // alternatively SPI_CPOL & ~SPI_CPHA, etc.
  ret = ioctl(fd, SPI_IOC_WR_MODE32, &tmp32);
  if (ret < 0) {
    perror("ioctl: SPI_IOC_WR_MODE32");
    return 1;
  }

  /* use the device as necessary here */

  close(fd);

  return 0;
}
```

## Using Spidev for TX/RX

Spidev is capable of both half-duplex and full-duplex SPI transfers. This can be
done by simply not specifying a TX __or__ RX buffer in a SPI transaction (though
of course at least one must be present). Also, both pointers may point to the
same buffer, allowing full-duplex reading and writing with only one buffer. This
example will write, then read, then perform a full-duplex transmission. Error
handling is omitted for brevity, so make sure to check for errors. SPI errors
are pretty rare, though, since there's no acknowledgment or handshaking with SPI
devices.

```c
/* pick up where we left off above, so 'fd' is our SPI device file descriptor */

// setup the SPI transfer structure to be used in the examples
struct spi_ioc_transfer m;
memset(&m, 0, sizeof(m));

/* half-duplex: write 5 bytes */
uint8_t txbuf[5] = { 0xAA, 0xBB, 0xCC, 0xDD, 0xEE };
m.tx_buf = (__u64)txbuf; // yes, they pass pointers as __u64 :(
m.rx_buf = 0;
m.len = 5;
ioctl(fd, SPI_IOC_MESSAGE(1), &m);

/* half-duplex: read 5 bytes */
uint8_t rxbuf[5] = {0};
m.tx_buf = 0;
m.rx_buf = (__u64)rxbuf;
m.len = 5;
ioctl(fd, SPI_IOC_MESSAGE(1), &m);

/* full-duplex: write 2 bytes, read 3 */
uint8_t txrxbuf[5] = { 0xAA, 0xBB, 0, 0, 0 };
m.tx_buf = (__u64)txrxbuf;
m.rx_buf = (__u64)txrxbuf;
m.len = 5;
ioctl(fd, SPI_IOC_MESSAGE(1), &m);
```

## Advanced Userspace SPI Features

While the above examples should suffice for most uses, there are some handy
features available that were not mentioned. The `spi_ioc_transfer` structure
contains a lot of useful flags and configuration options which are documented in
spidev.h. These allow overriding certain configuration options on a per-transfer
basis, such as bits per word, speed, and others. See spidev.h for the full suite
of options.

Additionally, the `SPI_IOC_MESSAGE` macro takes an argument which is the number
of messages to be transmitted. Above, we only used 1, as there was only one
message. However, you can pass entire arrays of messages to be transmitted, and
you can even specify whether or not the CS line should change after each message
using a flag in the `spi_ioc_transfer` structure. Here's an example sending
3 messages, receiving data on the last one, and only toggling the CS line after
the second and third transmissions. If all goes well, a logic analyzer should
show you something like this:

```
MOSI: _/[ 1 ]\_/[ 2 ]\___/[ 3 ]\_
MISO: _XXXXXXXXXXXXXXX___/[ 3 ]\_
CS:   \_______________/‾\_______/
```

Here's the code:

```c
struct spi_ioc_transfer msgs[3];
memset(msgs, 0, 3 * sizeof(*msgs));

for (int i = 0; i < 3; i++) {
  msgs[i].len = 5;
  msgs[i].tx_buf = /* buffer */;
}

// we don't have to set this for msgs[2], as it's the last transfer, so the
// deselect is implicit (which is why you don't have to set this to 1 if you're
// only sending one message)
msgs[1].cs_change = 1;

// receive data on the third packet
msgs[2].rx_buf = /* buffer */;

// finally, send our data
ioctl(fd, SPI_IOC_MESSAGE(3), msgs);
```

----

## References

1. [SPI userspace API (Linux kernel
   documentation)](https://www.kernel.org/doc/html/latest/spi/spidev.html)
