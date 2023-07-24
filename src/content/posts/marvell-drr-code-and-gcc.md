---
title: "Marvell DDR Code and GCC"
date: 2023-07-20T15:42:39-04:00
draft: false
---

I have a SolidRun MACCHIATObin Single Shot, and you can build and use UEFI on it. This involves compiling code from a variety of repos, one of those being [mv-ddr-marvell](https://github.com/MarvellEmbeddedProcessors/mv-ddr-marvell). To do this, I made a [git repo](https://github.com/AdrianKoshka/macc-uefi-build-script) full of helper scripts. I don't use the dev board much anymore, and it is mostly there to possibly help others build the collection of firmware that results in the image that I burn onto a microSD card.

## The Problem

So, I wouldn't be making a blog post on a blog I haven't posted on for years if I hadn't ran into some issue, and I did, between the aformentioned [Marvell DDR code](https://github.com/MarvellEmbeddedProcessors/mv-ddr-marvell) and recent versions of GCC (>=12). One of the last steps of building the firmware is building [TrustedFirmware-A](https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git), and getting some [Marvell Binaries](https://github.com/MarvellEmbeddedProcessors/binaries-marvell/tree/binaries-marvell-armada-SDK10.0.1.0), and that [DRR code](https://github.com/MarvellEmbeddedProcessors/mv-ddr-marvell.git) and throwing it all together in some way (I don't claim to understand the build process super well). My build environment is a Debian 12 VM running on Hyper V on the [Windows Dev Kit 2023](https://www.microsoft.com/en-us/d/windows-dev-kit-2023/94K0P67W7581). So what exactly happens? This happens:

```
$ bash tfa.sh 
make: Entering directory '/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a'

Building DRAM driver
  CC      apn806/mv_ddr_plat.c
  CC      apn806/mv_ddr_static.c
  CC      apn806/mv_ddr_validate.c
  CC      ddr_init.c
  CC      ddr3_init.c
  CC      ddr3_training_db.c
  CC      mv_ddr_build_message.c
  CC      mv_ddr_common.c
  CC      mv_ddr_spd.c
  CC      mv_ddr_mrs.c
  CC      mv_ddr_topology.c
  CC      mv_ddr4_training_db.c
  CC      drivers/mv_ddr_mc6.c
  CC      drivers/mv_ddr_xor_v2.c
  CC      ddr3_debug.c
  CC      ddr3_training.c
  CC      ddr3_training_bist.c
  CC      ddr3_training_centralization.c
  CC      ddr3_training_hw_algo.c
  CC      ddr3_training_ip_engine.c
  CC      ddr3_training_leveling.c
In file included from mv_ddr_atf_wrapper.h:232,
                 from ddr3_init.h:105,
                 from ddr3_training_leveling.c:98:
In function ‘mmio_read_32’,
    inlined from ‘mv_ddr_rl_dqs_burst’ at ddr3_training_leveling.c:1980:5:
/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/include/lib/mmio.h:46:16: error: array subscript 0 is outside array bounds of ‘volatile uint32_t[0]’ {aka ‘volatile unsigned int[]’} [-Werror=array-bounds]
   46 |         return *(volatile uint32_t*)addr;
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~
In function ‘mmio_read_64’,
    inlined from ‘mv_ddr_rl_dqs_burst’ at ddr3_training_leveling.c:1978:5:
/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/include/lib/mmio.h:56:16: error: array subscript 0 is outside array bounds of ‘volatile uint64_t[0]’ {aka ‘volatile long unsigned int[]’} [-Werror=array-bounds]
   56 |         return *(volatile uint64_t*)addr;
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~
cc1: all warnings being treated as errors
make[1]: *** [Makefile:473: /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/ble/ddr3_training_leveling.o] Error 1
make: *** [plat/marvell/armada/a8k/common/ble/ble.mk:35: /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/ble/mv_ddr_lib.a] Error 2
make: Leaving directory '/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a'
cp: cannot stat 'trusted-firmware-a/build/a80x0_mcbin/release/flash-image.bin': No such file or directory
```

`tfa.sh` is one of the helper scripts, and its contents are:

```
#!/usr/bin/env bash
BDIR=e2b

cd ${BDIR}
export WORKSPACE=$PWD
export PACKAGES_PATH=$PWD/edk2:$PWD/edk2-platforms:$PWD/edk2-non-osi

make -C trusted-firmware-a \
        PLAT=a80x0_mcbin \
        MV_DDR_PATH=$PWD/mv-ddr-marvell \
        SCP_BL2=$PWD/binaries/mrvl_scp_bl2.img \
        BL33=$PWD/Build/Armada80x0McBin-AARCH64/RELEASE_GCC5/FV/ARMADA_EFI.fd \
        all fip mrvl_flash
```

It sets a variable for our build directory, goes to that directory, and some others that I believe the build process expects, and then kicks off `make` for TF-A.

To drill down to the error more specifically:

```
In function ‘mmio_read_32’,
    inlined from ‘mv_ddr_rl_dqs_burst’ at ddr3_training_leveling.c:1980:5:
/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/include/lib/mmio.h:46:16: error: array subscript 0 is outside array bounds of ‘volatile uint32_t[0]’ {aka ‘volatile unsigned int[]’} [-Werror=array-bounds]
   46 |         return *(volatile uint32_t*)addr;
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~
In function ‘mmio_read_64’,
    inlined from ‘mv_ddr_rl_dqs_burst’ at ddr3_training_leveling.c:1978:5:
/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/include/lib/mmio.h:56:16: error: array subscript 0 is outside array bounds of ‘volatile uint64_t[0]’ {aka ‘volatile long unsigned int[]’} [-Werror=array-bounds]
   56 |         return *(volatile uint64_t*)addr;
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~
```

`gcc (Debian 12.2.0-14) 12.2.0` doesn't care for how Marvell decided to handle this array code, and I don't blame it, I'm not a firmware developer, but it looks a bit...weird to me. 


## The "Solution"

This started my quest of finding a version of `gcc` that would compile this code and not complain, and I started with `gcc` 7.5.0, but I eventually ended up finding out that `gcc` 11.4.0 is the latest version (as of the time I tried) that copmiled this code. After some googling, I found out how to [declare a toolchain](https://trustedfirmware-a.readthedocs.io/en/latest/getting_started/initial-build.html) for TF-A.

### Prep

Before going on to compiling `gcc` or `binutils`, you'll want to make sure you have some stuff installed:

```shell
$ sudo apt install build-essential acpica-tools device-tree-compiler uuid-dev libssl-dev gcc-aarch64-linux-gnu texinfo bison help2man --install-recommends -y
```

### Compiling GCC

This is easy enough, [there's documentation](https://gcc.gnu.org/wiki/InstallingGCC) on how to build it from source, and so I followed it.

```shell
alc@devvm:~/toolchain-src/$ wget https://ftp.gnu.org/gnu/gcc/gcc-11.4.0/gcc-11.4.0.tar.gz
alc@devvm:~/toolchain-src/$ tar xvf gcc-11.4.0.tar.gz
alc@devvm:~/toolchain-src/$ cd gcc-11.4.0
alc@devvm:~/toolchain-src/gcc-11.4.0$ ./contrib/download_prerequisites
alc@devvm:~/toolchain-src/$ cd ..
alc@devvm:~/toolchain-src/$ mkdir objdir
alc@devvm:~/toolchain-src/$ cd objdir
alc@devvm:~/toolchain-src/objdir$ $PWD/../gcc-11.4.0/configure --prefix=$HOME/buildroot/11.4.0 --enable-languages=c,c++,lto --enable-lto --disable-nls --disable-libquadmath --disable-libquadmath-support --enable-default-pie --disable-werror --enable-linker-build-id
alc@devvm:~/toolchain-src/objdir$ make -j5
alc@devvm:~/toolchain-src/objdir$ make install
```

Actually building `gcc` on my system takes a bit over half an hour, and I'll admit some of the enable/disable statements here aren't exactly scientific, but it's what made the process work for me.

### Compiling Binutils

This one is even more straight forward, and we need this for things like `ld` and `ar`.

```shell
alc@devvm:~/toolchain-src$ wget https://ftp.gnu.org/gnu/binutils/binutils-2.40.tar.xz
alc@devvm:~/toolchain-src$ tar xvf binutils-2.40.tar.xz
alc@devvm:~/toolchain-src$ cd binutils-2.40
alc@devvm:~/toolchain-src/binutils-2.40$ ./configure --prefix=$HOME/buildroot/11.4.0/ ./configure --prefix=$HOME/buildroot/11.4.0/ --enable-lto --enable-year2038 --with-pic --enable-deterministic-archives
alc@devvm:~/toolchain-src/binutils-2.40$ make -j5
alc@devvm:~/toolchain-src/binutils-2.40$ make install
```

### Modifying `tfa.sh`

As seen as in the [TF-A documentation](https://trustedfirmware-a.readthedocs.io/en/latest/getting_started/initial-build.html), we need to add `export CROSS_COMPILE=<path-to-aarch64-gcc>/bin/aarch64-none-elf-` to `tfa.sh` so that it will use our version of `gcc` and binutils, although I found I needed something more like `export CROSS_COMPILE=<path-to-aarch64-gcc>/bin/`, for me `tfa.sh` ended up looking like this:

```shell
#!/usr/bin/env bash
BDIR=e2b

cd ${BDIR}
export WORKSPACE=$PWD
export PACKAGES_PATH=$PWD/edk2:$PWD/edk2-platforms:$PWD/edk2-non-osi
export CROSS_COMPILE="$HOME/buildroot/11.4.0/bin/"

make -C trusted-firmware-a \
        PLAT=a80x0_mcbin \
        MV_DDR_PATH=$PWD/mv-ddr-marvell \
        SCP_BL2=$PWD/binaries/mrvl_scp_bl2.img \
        BL33=$PWD/Build/Armada80x0McBin-AARCH64/RELEASE_GCC5/FV/ARMADA_EFI.fd \
        all fip mrvl_flash

cp trusted-firmware-a/build/a80x0_mcbin/release/flash-image.bin uefi-mcbin-spi.bin
```

now if we run `tfa.sh` the final bit of output we get is:

```
Built /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/bl31.bin successfully

  OD      /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/bl31/bl31.dump
  HOSTCC  fiptool.c
  HOSTCC  tbbr_config.c
  HOSTLD  fiptool

Built fiptool successfully

Trusted Boot Firmware BL2: offset=0xD8, size=0x5321, cmdline="--tb-fw"
SCP Firmware SCP_BL2: offset=0x53F9, size=0x28A2C, cmdline="--scp-fw"
EL3 Runtime Firmware BL31: offset=0x2DE25, size=0xD253, cmdline="--soc-fw"
Non-Trusted Firmware BL33: offset=0x3B078, size=0x200000, cmdline="--nt-fw"

Built /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/fip.bin successfully


Built /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/boot-image.bin successfully

make[1]: Nothing to be done for 'all'.

Building flash image

Built /home/alc/macc-uefi-build-script/e2b/trusted-firmware-a/build/a80x0_mcbin/release/flash-image.bin successfully

make: Leaving directory '/home/alc/macc-uefi-build-script/e2b/trusted-firmware-a'
```

Hurray, `e2b/uefi-mcbin-spi.bin` is what we wanted in the end, and everything builds.