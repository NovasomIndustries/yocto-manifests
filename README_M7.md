# Yocto BSP for NOVAsom M7
## Build NOVAsom M7 Yocto BSP
### Prerequistes
Build of NOVASom M7 require a running Linux PC (or Linux Virtual Machine).  
Information about supported Linux distribution can be found [here](https://wiki.yoctoproject.org/wiki/Distribution_Support)

For each distribution you've to check that you have at least 50GB of free disk space and other prerequisites are met.  
You can find detailed prerequisites for each distribution [here](https://www.yoctoproject.org/docs/1.8/yocto-project-qs/yocto-project-qs.html)

### Initial build procedure
This procedure has to be used only the fisrt time that you install and build NOVAsom M7 Yocto BSP

	mkdir ~/bin
	curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
	chmod a+x ~/bin/repo
	PATH=${PATH}:~/bin

	cd <your_preferred_work_folder>
	mkdir bsp-novasom-m7
	cd bsp-novasom-m7
	repo init -u git@github.com:pagit2018/yocto-manifests.git -b master -m novasom-m7.xml
	repo sync
	TEMPLATECONF=../meta-novasom/conf/template/novasom-m7 source setup-environment build
	bitbake <image_name>
	
<image_name\> has to replaced with one of following:

* **m7-basic-image** (NOVAsom M7 custom image with basic configuration and commun utilities)
* **m7-wayland-image** (NOVAsom M7 custom image Wayland enabled with Weston implementation and QT support)

Or also general target image (see Yocto documentation about their usage):

* **core-image-minimal**
* **meta-toolchain**
* **meta-toolchain-sdk**
* **adt-installer**
* **meta-ide-support**

### Next build procedure
If you've already setup your initial build procedure, you can launch a new build with:

	cd <your_preferred_work_folder>
	cd bsp-novasom-m7
	source setup-environment build
	bitbake <image_name>

### SDcard preparation

When the Yocto build is completed you can find related output at path
	
	bsp-novasom-m7/build/tmp/deploy/images/novasom-m7-sd/	
	
And you can flash the image on a SDcard support with

	cd <your_preferred_work_folder>
	cd bsp-novasom-m7
	./make_sdcard.sh <image_name>
		
But before this you have to edit make_sdcard.sh script file adding the sdcard device name:

	DEVICE=/dev/<your_sdcard_device>
	
**NOTE: be careful to insert the correct sdcard device otherwise you could damage your linux system!!! If in doubt you can help with a cat /proc/partitions command**

## Usage instructions

### Device tree overlays
Modern linux kernels for embedded systems commonly use device tree for board configuration (i.e. multi-function pins configuration, peripherals configuration, ....).  
So it's quite common that you have to customize the board device-tree for your specific application needings.

Device tree customization can be done in two ways:

1. write a new application specific device tree (DTS format) and setup it into yocto machine file
1. use device tree overlay approach (see [here](https://www.kernel.org/doc/Documentation/devicetree/overlay-notes.txt)) 

NOVAsom M7 BSP has a base device tree (**rk3328-novasom-m7**) and some device tree overlay blobs that can bee applied during board boot up process (u-boot phase) on top of base device tree for configure some interfaces of NOVAsom M7 pin strip expansion connector.

Available device tree blobs are:

- **novasom-m7-i2c0**: configure a I2C master interface
    - pin3: SDA (was GPIO2_D1)
    - pin5: SCL (was GPIO2_D0)

- **novasom-m7-spi0**: configure a SPI master interface
    - pin19: SPI_TXD (was GPIO3_A1)
    - pin21: SPI_RXD (was GPIO3_A2)
    - pin23: SPI_CLK (was GPIO3_A0)
    - pin24: SPI_SS1 (was GPIO3_B0)
    - pin26: SPI_SS0 (was GPIO2_B4)
 
- **novasom-m7-uart1-2w**: configure a 2 wires UART interface 
    - pin15: UART1_TX (was GPIO3_A4)
    - pin18: UART1_RX (was GPIO3_A6)
 
- **novasom-m7-uart1-4w**: configure a 4 wires UART interface 
    - pin15: UART1_TX (was GPIO3_A4)
    - pin18: UART1_RX (was GPIO3_A6)
    - pin16: UART1_RTS (was GPIO3_A5)
    - pin22: UART1_CTS (was GPIO3_A7)

Standard provided yocto images boot up the board with the standard device tree (no overlays applied).  
You can apply one or more overlays in two ways:

1. editing *overlay.txt* file that you can find at *bsp-novasom-m7/sources/meta-novasom/recipes-bsp/u-boot-scr/files* path and rebuild the yocto image
1. editing u-boot environment:
    - use the M7 serial console and press a key during u-boot phase (stop boot and reach u-boot prompt)
    - create a *overlay* environment variable (type *edit overlay*)
    - add needed overlay files
    - save the environment using *saveenv* u-boot command
    - reboot the board

With both methods the syntax is (example):  
**overlay=novasom-m7-i2c0 novasom-m7-spi0 novasom-m7-uart1-2w**
 

### Read-only rootfs

Standard provided yocto images boot up the board with a read-only rootfs. 
This is a quite common approach for embedded systems based on SDCard or eMMC memory support, and ensure an optimal level of security and long term stability.

But of course this approch also put some restrictions; here some very basic information and suggestions.

Some folders of resulting rootfs are remapped to a RAM tmpfs (writable, but volatile, so lost after a reset), i.e.:
- **/tmp**
- **/var/log**
- **/var/lib**

**/var/log** contains all kernel / applications log files; you should point to this also for custom applications logs.

When you need to update the rootfs (i.e. adding some files or folders, or editing some configuration files), the main approach should be to customize your yocto image adding your application code, libraries, configurations files.... (see next chapter).  
But if you want to live modify the rootfs when already flashed and booted, you can safely temporarily remount the full rootfs in read/write mode with the command:

**mount -o remount,rw /**

After applying your changes you can remount in read-only mode with a:

**mount -o remount,ro /**

Both **m7-basic-image** and **m7-wayland-image** include also a separate writable data partition (256MB size) mounted as */data*, where you can store application non volatile data.

### eMMC support

M7 boards have a eMMC chip that can be used in place of a SD card for system boot and data storage.

For flashing contents on eMMC support the suggested procedure is:

- boot the M7 board with a SD card, using above instructions and one of supported images (**m7-basic-image** or **m7-wayland-image**)
- configure your yocto build for use **novasom-m7-emmc** machine; edit *build/conf/local.conf* and change to:  
	*MACHINE ?= 'novasom-m7-emmc'*
	
- launch:  
	*bitbake -c cleansstate u-boot-scr m7-basic-image*
	
	This will force a rebuild of these packages.  
	**NOTE:** this must be done each time you change the machine into your local.conf
	
- build the desired image for eMMC 
- copy the output image file to a USB stick. Your image file will be:

    - bsp-novasom-m7/build/tmp/deploy/images/novasom-m7-emmc/m7-basic-image-novasom-m7-emmc-gpt.img or
    - bsp-novasom-m7/build/tmp/deploy/images/novasom-m7-emmc/m7-wayland-image-novasom-m7-emmc-gpt.img
    
- insert USB stick on M7 board and use the M7 console (or ssh terminal)
- mount your USB stick with:  
	*mount /dev/sda1 /mnt*
	
- type:  
	*cd /opt/emmc*  
	*./make_emmc.sh /mnt/m7-basic-image-novasom-m7-emmc-gpt.img* (or adapt to used path)

- wait the end of flash procedure (it will take some time), than you can switch off your M7 board and remove the SD card: at next boot it shall boot from the eMMC device

**NOTE:** SD card has a boot precedence on eMMC: if valid boot partitions are present both on inserted SD card and eMMC device, the M7 board will boot from SD card (this allow you to recover unwanted situations and re-flash the eMMC content). 

### Image customization

You can find on web a lot of instruction and guides on general yocto usage and image customization process. Some useful links:

- [Yocto Quick Start](https://www.yoctoproject.org/docs/2.0/yocto-project-qs/yocto-project-qs.html)
- [Yocto Reference Manual](https://www.yoctoproject.org/docs/latest/ref-manual/ref-manual.html)
- [Yocto Development Manual](https://www.yoctoproject.org/docs/1.6.1/dev-manual/dev-manual.html)

Here you can find some information about NOVAsom M7 Yocto BSP and their customization

A Yocto BSP project is always organized by layers; the better approach for a BSP customization is to add your own layer on top of available layer structure and ad assign to it the highest priority.

With provided BSP the current higher priority is 30, assigned to *meta-novasom-layer* (see layer.conf). Your customization layer shall have a higher priority.

This will allow you to easily integrate available updates on standard layer without the risk of overwriting your contents.

**Yocto machine**

NOVAsom M7 Yocto BSP provided machines can be found at *bsp-novasom-m7/sources/meta-novasom/conf/machine* path.  
Supported machine for this BSP are:

- **novasom-m7-sd** (for images to be installed on SD card support)
- **novasom-m7-emmc** (for images to be flashed onto M7 onboard eMMC)

The selected machine has to be definded into your local.conf file (BSP default is **novasom-m7-sd**).


**U-boot**

U-boot customizations should be added to a **u-boot-rockchip_%.bbappend** into your own layer.  
See *bsp-novasom-m7/sources/meta-novasom/recipes-bsp/u-boot/u-boot-rockchip_20171218.bbappend* as example.

**Linux kernel**

Linux kernel customizations should be added to a **linux-rockchip_4.4.bbappend** into your own layer.  
See *bsp-novasom-m7/sources/meta-novasom/recipes-kernel/linux/linux-rockchip_4.4.bbappend* as example.

Here you can can change the kernel configuration (**defconfig**) and/or add a new board device tree.

**Device tree overlays**

Device tree overlay customizations should be added to a **dt-overlay_%.bbappend** into your own layer.  
See *bsp-novasom-m7/sources/meta-novasom/recipes-kernel/dt-overlay/dt-overlay_0.1.bb* as example.

**Image customization**

Yocto image customizations should be made by creating a **your_name-image.bb** into your own layer.  
See *bsp-novasom-m7/sources/meta-novasom/recipes-core/images/m7-basic-image.bb* as example.

You can also create into your layer *.bbappend* to customize existing images (i.e. *m7-basic-image.bbappend*)

With the image customizazation you can add standard contents (recipes) to your images (i.e. a web server, an ftp client, libraries, python, ....) and/or add your own recipes that contains yow application software. 

---

Maintainer: Geninatti Paolo <tech@pagit.eu>
