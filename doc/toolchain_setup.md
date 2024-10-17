# Toolchain setup

**NOTE:** Documentation migrated to dev.tillitis.se, this is kept for
history. This is likely to be outdated.

Here are instructions for setting up the tools required to build the
project. Tested on Ubuntu 22.10.

## General development environment

The following is intended to be a complete list of the packages that
are required for doing all of the following:

 - building and developing [TKey device and client
   apps](https://github.com/tillitis/tillitis-key1-apps)
 - building our [QEMU machine](https://github.com/tillitis/qemu/tree/tk1)
   (useful for apps dev)
 - building and developing firmware and FPGA gateware (which also
   requires building the toolchain below)

```
sudo apt install build-essential clang lld llvm bison flex libreadline-dev \
                 gawk tcl-dev libffi-dev git mercurial graphviz \
                 xdot pkg-config python3 libftdi-dev \
                 python3-dev libeigen3-dev \
                 libboost-dev libboost-filesystem-dev \
                 libboost-thread-dev libboost-program-options-dev \
                 libboost-iostreams-dev cmake libusb-1.0-0-dev \
                 ninja-build libglib2.0-dev libpixman-1-dev \
                 golang clang-format \
                 gcc-arm-none-eabi libnewlib-arm-none-eabi \
                 libstdc++-arm-none-eabi-newlib
```

## Device permissions

To allow sudo-less programming, you can install a udev rule that will
assign the tkey programmer, and also an unprogrammed CH552, to the
dialout group. You will also need to add your user to this group:

```
sudo cp contrib/99-tillitis.rules /etc/udev/rules.d
sudo udevadm control --reload-rules
sudo usermod -aG dialout ${USER}
```

To apply the new group, log out and then log back in.

You can check the device permissions to determine if the group was
successfully applied. First, use lsusb to find the location of the
programmer:

```
lsusb -d 1209:8886
Bus 001 Device 023: ID 1209:8886 Generic TP-1
```

Then, you can check the permissions by using the bus and device
number reported above. Note that this pair is ephemeral and may
change after every device insertion:

```
ls -l /dev/bus/usb/001/023
crw-rw---- 1 root dialout 189, 22 Feb 16 14:58 /dev/bus/usb/001/023
```

## Gateware: Yosys/Icestorm toolchain

If the LED of your TKey is steady white when you plug it, then the
firmware is running and it's already usable! If you want to develop
TKey apps, then only the above general development environment is
needed.

Compiling and installing Yosys and friends is only needed if your TKey
is not already running the required firmware and FPGA gateware, or if
you want to do development on these components.

These steps are used to build and install the
[icestorm](http://bygone.clairexen.net/icestorm/) toolchain. The
binaries are installed in `/usr/local`. Note that if you have or
install other versions of these tools locally, they could conflict
(case in point: `yosys` installed on MacOS using brew).

    git clone https://github.com/YosysHQ/icestorm
    cd icestorm
    git checkout d20a5e9001f46262bf0cef220f1a6943946e421d
    make -j$(nproc)
    sudo make install
    cd ..

    # Custom iceprog for the RPi 2040-based programmer (will be upstreamed).
    # Note: install dependencies for building tillitis-iceprog on Ubuntu:
    # sudo apt install libftdi-dev libusb-1.0-0-dev
    git clone -b interfaces https://github.com/tillitis/icestorm tillitis--icestorm
    cd tillitis--icestorm/iceprog
    make
    sudo make PROGRAM_PREFIX=tillitis- install
    cd ../..

    git clone https://github.com/YosysHQ/yosys
    cd yosys
    git checkout yosys-0.26
    make -j$(nproc)
    sudo make install
    cd ..

    git clone https://github.com/YosysHQ/nextpnr
    cd nextpnr
    git checkout nextpnr-0.5
    cmake -DARCH=ice40 -DCMAKE_INSTALL_PREFIX=/usr/local .
    make -j$(nproc)
    sudo make install
    cd ..

References:
* http://bygone.clairexen.net/icestorm/

## Firmware: riscv toolchain

The TKey implements a [picorv32](https://github.com/YosysHQ/picorv32)
soft core CPU, which is a RISC-V microcontroller with the C
instructions and Zmmul extension, multiply without divide
(RV32ICZmmul). You can read
[more](https://www.sifive.com/blog/all-aboard-part-1-compiler-args)
about it.

The project uses the LLVM/Clang suite and version 15 or later is
required. As of writing Ubuntu 22.10 has version 15 packaged. You may
be able to get it installed on older Ubuntu and Debian using the
instructions on https://apt.llvm.org/ . There are also binary releases
here: https://github.com/llvm/llvm-project/releases

References:
* https://github.com/YosysHQ/picorv32

If your available `objcopy` and `size` commands is anything other than
the default `llvm-objcopy` and `llvm-size` define `OBJCOPY` and `SIZE`
to whatever they're called on your system before calling `make`.

## CH552 USB to Serial firmware

The USB to Serial firmware runs on the CH552 microcontroller, and
provides a USB CDC profile which should work with the default drivers
on all major operating systems. MTA1-USB-V1 and TK-1 devices come
with the CH552 microcontroller pre-programmed.

Toolchain setup and build instructions for this firmware are detailed
in the
[ch552_fw directory](../hw/usb_interface/ch552_fw/README.md)
