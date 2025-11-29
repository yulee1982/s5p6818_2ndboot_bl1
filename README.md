2nd bootloader for Nexell s5pxx18 series SoC
============================================
This README contains information about Nexell s5pxx18 series Soc's
2nd bootloader source. 

What is 2nd bootloader?
----------------------
Rom Boot loads 2nd bootloader from non volatile storage to internal SRAM 
and jump to it, and 2nd bootloader does cpu initialization and set up,
and the memory initialization. In the end, 2nd bootloader loads 3rd bootloader
(u-boot or ARM ATF so on) from non volatile storage to DRAM and jump to it.

Description of Important files
------------------------------
StartUp Entry
 - Rom Boot jump to here
File Path : src/startup_arch{XX}.S

C Main Code File
 - after stack set up, Assem code jump to BootMain entry 
File Path : src/secondboot.c

Power Manager IC 
 - PMIC Control code for ARM & DDR Voltage
File Path : src/pmic.c

DDR3 Memory Init
 - DDR3 Memory Initialize Code.
File Path : src/init_DDR3.c

LPDDR3 Memory Init
 - LPDDR3 Memory Initialize Code.
File Path : src/init_LPDDR3.c

Config File
 - Global Defines and compiler setting
File Path : config.mak


Source Tree
-----------
|--src
|--prototype
   base
   module
|--nsih-generator
|--refernce-nsih
 
    
How to build
------------
Get Toolchain
  # wget https://releases.linaro.org/14.04/components/toolchain/binaries/gcc-linaro-aarch64-none-elf-4.8-2014.04_linux.tar.bz2
  # tar xvjf gcc-linaro-aarch64-none-elf-4.8-2014.04_linux.tar.bz2
  # export PATH={gcc-linaro-path}/bin:$PATH

Fix config.mak
 - select OPMODE
 - select BOARD
 - set CROSS_TOOL_TOP: {gcc-linaro-aarch64-path}/bin

Build
   # make

Other Toolchain
aarch64-linux-gnu-gcc -v
gcc 6.4.1 20170707 (Linaro GCC 6.4-2017.08)
-----------------------------------------------------------------------------------
## Boot loader for Nanopc T3 Plus

First stage boot loader, loaded by s5p6818 boot ROM. Loads u-boot.

### Compilation

A gcc cross-compiler is needed. I'm using gcc-linaro 6.x available on Ubuntu,
in _gcc-aarch64-linux-gnu_ package. To build, run _make_.  The _make_ command builds
_out/bl1-nanopi.bin_ binary. This is the _bl1_ binary for NanoPi.

### Boot process on Nanopc T3 Plus

Kernel loading on this board is performed in a few stages, namely:

 * S5P6818 boot ROM loads image, a first stage loader, to static RAM. The RAM is named "static" in opposite to "dynamic" RAM used as the main memory of the device. Dynamic RAM needs some initialization before use. Static RAM does not need any initialization. It is available immediately and it is the only memory available at startup. This board has only 64 kilobytes of static RAM. The static memory starts at address 0xffff0000 and ends at 0xffffffff. Boot ROM loads the first stage loader at address 0xffff0000.
 * First stage loader initializes main memory and then loads u-boot to main memory. The u-boot is loaded at address chosen by programmer. The 32-bit u-boot provided by FriendlyARM should be loaded at address 0x42c00000. On the other hand, u-boot for Samsung ARTIK (from which the [u-boot-nanopi-m3](https://github.com/rafaello7/u-boot-nanopi-m3) is forked) should be loaded at address 0x43c00000.
 * The u-boot loads Linux kernel and initrd image (initial ramdisk) to memory and boots the kernel. On arm64 platform also a _device tree_ image must be loaded. The _device tree_ image contains information about the SOC layout: what hardware it contains and how the hardware components are connected together. More information about device tree may be found at [elinux.org](http://elinux.org/Device_Tree_What_It_Is).

The images are normally loaded from SD card. But NanoPi may be also connected to PC USB port and all the images may be loaded using USB connection. The device must be plugged through micro-usb port, such used for powering the device. Images may be loaded using [nanopi-load](https://github.com/rafaello7/nanopi-load) tool.

The _nanopi-load_ tool needs _libusb-1.0_ library. To compile the tool on _Debian_ or _Ubuntu_, _libusb-1.0-0-dev_ and _pkg-config_ packages should be installed.

//################################################################
To update bl1 and uboot on Nanopc T3 Plus.
In NSIH.txt or .../2ndboot_bl1/bl1-nanopi-m3-master/src/startup_aarch64.S
// Main Bootloader Load Info
00010200	// 0x040 : Device Read Address from 2ndboot Device.  0x10200/512=129
00040000	// 0x044 : Load Size for 2ndboot.
FFFF0000	// 0x048 : Load Address for 2ndboot.
FFFF0000	// 0x04C : Launch Address for 2ndboot.
00000000	// 0x050	[7:0] - Channel number
03000000	// 0x054	Boot From SDMMC	[31:24] - 0: USB, 1: SPI, 2: NAND, 3: SDMMC, 4: SDFS
00000000	// 0x058

1.Create file xxx_link_bin.sh
#!/bin/sh

#_2ndbootnsih64=NSIH64.bin   (not use, NSIH64 has been written into startup_aarch64.S)
_2ndboot=bl1-nanopc_t3.bin
#_3rdbootnsih64=NSIH64_uboot.bin  (not use, NSIH64 for uboot head has been written into start.S)
                                                         # .../u-boot-friendlycore_master/arch/arm/cpu/armv8/start.S
uboot=u-boot.bin

target=_2ndboot_uboot.bin

#create target bin file
dd if=/dev/zero of=${target} bs=512 count=1500
#copy first bin (nish head bin) into targer bin file
#dd if=${_2ndbootnsih64} of=${target} bs=512 seek=0 conv=notrunc
#copy 2ndboot.bin into targer.bin
dd if=${_2ndboot} of=${target} bs=512 seek=0 conv=notrunc
#copy nish.bin into targer.bin file
#dd if=${_3rdbootnsih64} of=${target} bs=512 seek=128 conv=notrunc
#copy uboot.bin into targer.bin file
dd if=${uboot} of=${target} bs=512 seek=128 conv=notrunc
echo "OK"
#end of xxx_link_bin.sh

Place bl1-nanopc_t3.bin, u-boot.bin, and xxx_link_bin.sh in the same directory, generate _2ndboot_uboot.bin.
sudo ./xxx_link_bin.sh

2.Inject binary file. Create file xxx_fuse_sd.sh
#!/bin/sh

#USAGE()
#{
#    echo Usage: $(basename "$0") '<device> <bootloader>'
#    echo '       device     = disk device name for SD card.'
#    echo '       bootloader = /home/xxx/work/aarch64/_2ndboot_uboot.bin'
#    echo 'e.g. '$(basename "$0")' /dev/sdb _2ndboot_uboot.bin'
#}

#[ -z "$1" -o -z "$2" ] && { USAGE; exit 1; }

#dev="$1"
#xboot="$2"  #use this = sudo ./xxx_fuse_sd.sh /dev/sdb _2ndboot_uboot.bin
dev=/dev/sdb
xboot=_2ndboot_uboot.bin

[ -b "${dev}" ] || { echo "${dev} is not a valid block device"; exit 1; }
[ X"${dev}" = X"${dev%%[0-9]}" ] || { echo "${dev} isa partition, please use device, perhaps ${dev%%[0-9]}"; exit 1; }
[ -f ${xboot} ] || { echo "${xboot} is not a bootloader binaryfile."; exit 1; }

dd if="${xboot}" of="${dev}" bs=512 seek=1 conv=sync

sync;

echo "OK"
#end of xxx_fuse_sd.sh

sudo ./xxx_fuse_sd.sh /dev/sdb _2ndboot_uboot.bin

/dev/sdb must be changed to your device.
[sdmmc] |Unallocated (use for boot)   | Device partition (sdb)                |
