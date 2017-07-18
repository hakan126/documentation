# Instructions for Dynamically Loading Device Tree Overlays into Linux Kernel

This document provides instructions for dynamically loading the device tree overlays (dtbo) into
linux kernel running on DragonBoard410c.

# Table of Contents

- [1) Device Tree Compiler](#1-device-tree-compiler)
    - [1.1) Installing the Compiler](#11-installing-the-compiler)
- [2) Enable Overlay Support in Kernel](#2-enable-overlay-support-in-kernel)
    - [2.1) Cloning and Building the kernel](#21-cloning-and-building-the-kernel)
- [3) Load Overlays Dynamically](#3-add-overlays-dynamically)
    - [3.1) Compiling the Overlays](#31-compiling-the-overlays)
    - [3.2) Loading Overlays via Configfs](#32-inserting-overlays-via-configfs)
     
 ***
 
 # 1) Device Tree Compiler
 
 First of all we need to install device tree compiler (dtc) for compiling the source files (dts) into
 overlays (dtbo). Overlay support was added to the mainline dtc by v1.4.2 only. But, the one which is available
 as a package in debian based distros is v1.4.0. So, we need to install the compiler from source.
 
 ## 1.2 Installing the Compiler
 
```shell
$ sudo apt-get install flex bison swig
$ git clone git://git.kernel.org/pub/scm/utils/dtc/dtc.git
$ cd dtc
$ make
$ sudo make install PREFIX=/usr
```

# 2) Enable Overlay Support in Kernel

Next, overlay support needs to be enabled in the kernel. For which a custom kernel would be used other than
the one available in Qualcomm landing page.

## 2.1 Cloning and Building the kernel

```shell
$ export ARCH=arm64
$ export CROSS_COMPILE=<path to your GCC cross compiler>/aarch64-linux-gnu-
$ git clone -b configfs-overlay https://github.com/Mani-Sadhasivam/linux-qcom.git
$ cd linux-qcom
$ make defconfig distro.config
$ make -j4 Image dtbs
$ make -j4 modules
$ make modules_install INSTALL_MOD_STRIP=1 INSTALL_MOD_PATH=<folder>
```
> Note: Replace <folder> with the path of folder to put kernel modules. The built kernel modules needs to be transferred 
to the root file system, under ***/lib/modules*** folder.

For further instructions on how to create bootable image and flashing onto Dragonboard410c, refer release notes 
[here](http://builds.96boards.org/releases/dragonboard410c/linaro/debian/latest/)

# 3) Load Overlays Dynamically

After flashing the patched kernel onto Dragonboad410c, following instructions allows to load device tree overlays
dynamically through configfs.

## 3.1 Compiling the Overlays

There are some example overlays available for reference. You can modify them according to the preferred device. Below
example shows compiling the overlays for i2c based **TSYS01** and **MS8607** sensors. Device drivers for the appropriate 
sensors should have device tree support enabled for allowing cold plug.

```shell
$ git clone https://github.com/Mani-Sadhasivam/DT-Overlays.git
$ cd DT-Overlays
$ make
```

After successful compilation, device tree blobs (dtbo) will be available in ***bin*** directory.

## 3.2 Loading Overlays via Configfs

Now its the time to insert the device tree blobs into running kernel using configfs.

```shell
$ sudo su
# mount -t configfs none /sys/kernel/config
```
When configfs has been mounted properly, that directory should have been populated with subdirectories 
***/sys/kernel/config/device-tree/overlays***

```shell
# mkdir -p /sys/kernel/config/device-tree/overlays/tsys01
# cd <bin directory under DT-Overlays>
# cat tsys01.dtbo > /sys/kernel/config/device-tree/overlays/tsys01/dtbo
```

After loading, the device should appear under ***/proc/device-tree/soc/i2c@78b6000/***

That't it! You have loaded device tree overlay dynamically. But this wont be sufficient, you need to load your device driver
also to work with the device. By this time, if the driver has been compiled into the kernel (by selecting *y* during **make
 menuconfig**), then the driver should have been probed successfully and it will appear under ***/sys/bus/i2c/devices/***
 
If the driver was compiled as a kernel module, then insert it using **modprobe**
