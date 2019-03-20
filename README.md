# Perfctr-Xen

Copyright (C) 2011, 2013 Ruslan Nikolaev

**_Note: It was last released in 2013; I mirrored it here on GitHub._**

The preferable version is the recent beta version (from 2013) of framework which requires patching of three system components: [Xen 4.2.0](https://downloads.xenproject.org/release/xen/4.2.0/xen-4.2.0.tar.gz), [Linux 3.2.30](https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.2.30.tar.xz) guest, and [perfctr 2.6.42.2](https://github.com/rusnikola/perfctr-xen/perfctr-beta/perfctr-2.6.42.2) (my modified version of 2.6.42 which can work with Linux 3.2.30). Note that this is a beta version (which may still contain bugs). Patches for this version are in _perfctr-beta_.

Note that the above perfctr version contains patches for various (including more recent) versions of Linux. Older versions will most likely not work. More recent versions may or may not work, and they are not tested; it is not recommended to use anything other than 3.2.30.

Previous stable version (from 2011) of framework (which is now obsolete due to its reliance on the PvOps 2.6.x kernel) requires patching of:
[Xen 4.0.1](https://downloads.xenproject.org/release/xen/4.0.1/xen-4.0.1.tar.gz), PvOps Linux 2.6.32.21 guest, and [perfctr 2.6.42](http://user.it.uu.se/~mikpe/linux/perfctr/2.6/perfctr-2.6.42.tar.xz) driver. Follow the installation instructions below to properly set up the framework. Also, make sure that Linux guest you use is of 2.6.32-pvops branch. (It may be necessary to switch it if the default branch is 2.6.31 as described here.) Patches for this version are in _perfctr-obsolete_.

Instructions below are for the older stable version. For the beta
version, please adjust corresponding names of directories and patches. For the beta version, there may be additional steps mentioned in instructions.

## Publication

[Paper](https://rusnikola.github.io/files/vee2011.pdf)

**Perfctr-Xen: a framework for performance counter virtualization.** Ruslan Nikolaev and Godmar Back. In Proceedings of the 7th ACM SIGPLAN/SIGOPS international conference on Virtual Execution Environments (VEE'11), pp.15-26. Newport Beach, CA, USA.

This material is based upon work supported by the National Science
Foundation under Grant No. [0720673](https://www.nsf.gov/awardsearch/showAward?AWD_ID=0720673).

_Any opinions, findings, and conclusions or recommendations expressed in
this material are those of the author(s) and do not necessarily reflect the
views of the National Science Foundation._

## Installation

Download from this website the latest version of patches for the corresponding versions of Linux/Xen/Perfctr mentioned above.

Download and extract Xen, Linux and perfctr. (PvOps Linux 2.6.32.21: it may be necessary to download the Linux archive manually as opposed to automatic download during compilation. Follow the instructions provided here. Make sure to put the downloaded Linux files under standard location xen-4.0.1/linux-2.6-pvops.git ; for this, you need to rename the downloaded directory and move into xen-4.0.1.)  For the newer beta version and Xen 4.2/Linux 3.2.30, Linux is compiled separately from Xen, and you can place Linux elsewhere but make sure to follow integration instructions; please also choose configuration options as specified in step 5.
Patch all of them

```
cd xen-4.0.1
patch -p1 < ../xen_r1.patch
cd linux-2.6-pvops.git
patch -p1 < ../../linux_r1.patch
cd ../../perfctr-2.6.42
patch -p1 < ../perfctr_r1.patch
```

* Integrate perfctr into Linux

```
cd xen-4.0.1/linux-2.6-pvops.git
patch -p1 < ../../perfctr-2.6.42/patches/patch-kernel-2.6.32
cp -r ../../perfctr-2.6.42/linux/* .
```

* For the newer beta version and Xen 4.2 / Linux 3.2.30, Linux is compiled separately from Xen. You need to follow traditional Linux compilation procedure (i.e., make, make menuconfig, make install) choosing the following options: (Disable CONFIG\_PERFCTR\_DEBUG and CONFIG\_PERFCTR\_INIT\_TESTS!)

```
CONFIG_PERFCTR=y
CONFIG_KPERFCTR=y
CONFIG_PERFCTR_DEBUG=n
CONFIG_PERFCTR_INIT_TESTS=n
CONFIG_PERFCTR_VIRTUAL=y
CONFIG_PERFCTR_GLOBAL=y
CONFIG_PERFCTR_INTERRUPT_SUPPORT=y
CONFIG_PERFCTR_CPUS_FORBIDDEN_MASK=y
CONFIG_PERFCTR_WIDECTR=y
CONFIG_PERFCTR_XEN=y
```

* Compile & install Xen and Linux guest. (Perfctr support should be enabled automatically after applying patches above. For the newer beta version it is enabled after you select corresponding options in Linux, step 5.)

```
make
make install
```

* Follow the standard instructions for creating initrd image and adding Xen to GRUB. It can be found at xen-4.0.1/README. Make sure that the hypervisor line you add to the GRUB configuration contains an additional perfctr option, so that perfctr-xen support will be enabled. (e.g. multiboot /boot/xen-4.0.gz dummy=dummy dom0\_mem=4096M perfctr)

* Copy perfctr-2.6.42 to perfctr-2.6.x (only if you intend to use PAPI).

```
cp -r perfctr-2.6.42 perfctr-2.6.x
```

* Compile & install perfctr user library

```
cd perfctr-2.6.42
make
make install
```

* If you want to use PAPI or anything that works on top of PAPI (HPCToolkit, PerfExplorer, etc.), download the latest version from here. The code is currently known to work with PAPI 4.1.0-4.4.0

* Remove PAPI's copy of perfctr-2.6.x and perfctr-2.7.x and replace it with the patched one

```
cd papi-4.1.0/src
rm -r perfctr-2.6.x perfctr-2.7.x
mv ../../perfctr-2.6.x .
```

* Compile & install PAPI (if desired)

```
./configure --prefix=/usr
make
make install
```

* The compiled kernel is available at /boot. It will be used by Dom0 when you reboot the computer and load Xen hypervisor. Since domU domains (Dom1, Dom2, etc.) also need to use perfctr-xen, you can just copy the compiled kernel, initrd, modules to the corresponding domain image. (This is possible because the compiled kernel contains both Dom0 and DomU support. It even works with HVM domains.)

## Testing

After rebooting, you may want to test the installation.

* Make sure that the perfctr device is available at /dev/perfctr.

* Run perfex test to test a-mode counters (You will get 0 here if you forgot to enable perfctr option in GRUB.) By default, you will only see TSC. You can also specify any supported hardware event (-e option) but the event type depends on your hardware, and subsequently no example is presented here. PAPI tests (see below) do not require you to know the actual event number.

```
root@ubuntu:~# perfex true
tsc				            5695128
```

* After this you can test i-mode counters for which a special signal test exists. You should get something similar to this.

```
root@ubuntu:~/perfctr-2.6.42/examples/signal# ./signal
Control used:
tsc_on			1
nractrs			0
nrictrs			2
pmc_map[0]		0
evntsel[0]		0x0051FF10
ireset[0]		-25
pmc_map[1]		1
evntsel[1]		0x005100C4
ireset[1]		-25

on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
on_sigio: PMC overflow set 0x2 at pc 0x7fd5d6a4aa47
...
on_sigio: PMC overflow set 0x3 at pc 0x7fd5d6a4aa47
```

* If you installed PAPI, you can run some built-in tests.

The following test will affect a-mode counters:
```
root@ubuntu:~/papi-4.1.0/src/ctests# ./zero
Test case 0: start, stop.
-----------------------------------------------
Default domain is: 1 (PAPI_DOM_USER)
Default granularity is: 1 (PAPI_GRN_THR)
Using 20000000 iterations of c += a*b
-------------------------------------------------------------------------
Test type    : 	           1
PAPI_FP_INS  : 	     59998672
PAPI_TOT_CYC : 	    201096584
Real usec    : 	        84198
Real cycles  : 	    190371459
Virt usec    : 	        84195
Virt cycles  : 	    190363923
-------------------------------------------------------------------------
Verification: none
zero.c                                   PASSED
```

The following test will affect i-mode counters:
```
root@ubuntu:~/papi-4.1.0/src/ctests# ./overflow
...
handler(0 ) Overflow at 0x40282c! bit=0x2 
handler(0 ) Overflow at 0x402813! bit=0x2 
handler(0 ) Overflow at 0x40283f! bit=0x2 
Test case: Overflow dispatch of 2nd event in set with 2 events.
---------------------------------------------------------------
Threshold for overflow is: 1000000
Using 20000000 iterations of c += a*b
-----------------------------------------------
Test type    :                1               2
PAPI_FP_INS  :         59999127        59998329
PAPI_TOT_CYC :        201339533       202176272
Overflows    :                               59
-----------------------------------------------
Verification:
Row 1 approximately equals 40000000 40000000
Column 1 approximately equals column 2
Row 3 approximately equals 59 +- 75 %
Overflows: total(59) > max(104) || total(59) < min(14) overflow.c                               PASSED
```

## Caution

Do not use perfctr option (in GRUB configuration) and vpmu simultaneously. The latter one enables hardware counters virtualization in HVM mode, and it should never be used with the modified perfctr.

