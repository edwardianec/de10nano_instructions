Open up a terminal and execute embedded_command_shell.sh in your SoC EDS installation
folder `~/tools/intelFPGA_lite/18.1/embedded/embedded_command_shell.sh ` After executing
that shell script, you now have access to several tools of SoC EDS which we will need in
next steps.

# Custom IP
Append this to the end of your custom platform designer tcl script. This helps to
describe a node in linux device tree. <br>
Example for custom IP.

```
# Device tree generation
set_module_assignment embeddedsw.dts.vendor "skb"
set_module_assignment embeddedsw.dts.compatible "dev, streaming_source"
set_module_assignment embeddedsw.dts.group "streaming_source"
```

# Getting and setting up the GCC toolchain
To compile U-Boot and the Linux kernel, we need a GCC toolchain for the ARMv7 instruction
set. Download *gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf* and unpack it.
Open /home/edward/.bashrc and add in the end the path to linaro gcc.<br>
```export CROSS_COMPILE="/home/edward/code/projects/fpga/de10nano/docs/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-"```<br>
save and reopen terminal

# RBF && Header file

To flash FPGA from UBOOT or Linux we need to convert .sof file to .rbf.<br>
```quartus_cpf -c -o bitstream_compression=on DE10_NANO_SoC_GHRD.sof soc_system.rbf```<br>
FPGA Peripherals can be seen in ARM throw AXI bridges using addresses described in quartus
platform designer. 
```bash
sopc-create-header-files soc_system.sopcinfo --single software/hps_0.h --module
hps_0
```
This command creates header file with macros.<br>

# Preloader (SPL)

In your terminal, go to your DE10_NANO_SoC_GHRD folder and execute:<br>
```bsp-editor &```<br>
which will open up the BSP Editor. Click on ***File → New HPS BSP*** and select the
correct Preloader settings directory: DE10_NANO_SoC_GHRD/hps_isw_handoff/soc_system_hps_0/
folder. The rest of the settings will get filled in for you, so just click OK:

In the left panel, click on Common, and enable **FAT_SUPPORT**.<br>
That is the only change we had to make, so click Generate button in the bottom right of the BSP Editor window. When it’s done, click Exit.

## Compiling the preloader
The generated output of the BSP Editor can be found in the DE10_NANO_SoC_GHRD/software/spl_bsp/ folder. Also generated is a Makefile that we can use to compile the preloader. <br>
``cd software/spl_bsp; make``<br>
Generated preloader-mkpimage.bin is the file we need
later during creating sdcard image.

# UBOOT
https://github.com/zangman/de10-nano/blob/master/docs/Building-the-Universal-Bootloader-U-Boot.md<br>
https://bitlog.it/20170820_building_embedded_linux_for_the_terasic_de10-nano.html

## Getting and compiling U-Boot
Altera/Intel have their own version of U-Boot on the Altera opensource GitHub repository, which is modified to run specifically on their FPGAs. Let’s clone their repository:

```bash
git clone https://github.com/altera-opensource/u-boot-socfpga.git
cd u-boot-socfpga
git checkout rel_socfpga_v2013.01.01_17.08.01_pr
```

To make sure that we start with a clean slate before compiling, we will clean the U-Boot folder:
```bash
make mrproper
```
Now let's assign MAC address. To do this, open the following file in a text editor. I am using nano but you can use vim, gedit or any other editor:

```bash
nano include/configs/socfpga_common.h
```
Scroll down to the section that has the following lines:
```c
#ifndef CONFIG_EXTRA_ENV_SETTINGS
#define CONFIG_EXTRA_ENV_SETTINGS \
        "fdtfile=" CONFIG_DEFAULT_FDT_FILE "\0" \
        "bootm_size=0xa000000\0" \
        "kernel_addr_r="__stringify(CONFIG_SYS_LOAD_ADDR)"\0" \
        "fdt_addr_r=0x02000000\0" \
        "scriptaddr=0x02100000\0" \
        "pxefile_addr_r=0x02200000\0" \
        "ramdisk_addr_r=0x02300000\0" \
        "socfpga_legacy_reset_compat=1\0" \
        BOOTENV

#endif
```
Modify this to add the U-Boot environment variable ethaddr as shown below. Don't forget the \0 at the end as well as the \ or it won't work.
```c
#ifndef CONFIG_EXTRA_ENV_SETTINGS
#define CONFIG_EXTRA_ENV_SETTINGS \
        "fdtfile=" CONFIG_DEFAULT_FDT_FILE "\0" \
        "bootm_size=0xa000000\0" \
        "kernel_addr_r="__stringify(CONFIG_SYS_LOAD_ADDR)"\0" \
        "fdt_addr_r=0x02000000\0" \
        "scriptaddr=0x02100000\0" \
        "pxefile_addr_r=0x02200000\0" \
        "ramdisk_addr_r=0x02300000\0" \
        "socfpga_legacy_reset_compat=1\0" \
        "ethaddr=56:6b:20:e9:4a:47\0" \
        BOOTENV

#endif
```
And then we will compile U-Boot:

```bash
make socfpga_cyclone5_config
make socfpga_cyclone5
cd ..
```
This produces the u-boot.img U-Boot image file in the same folder.

## Setting up the boot script
U-Boot needs a boot script to know how to setup the FPGA and load the Linux kernel. Create a file named boot.script in the DE10_NANO_SoC_GHRD/software/ folder with the following contents:
```bash
echo ## God bless your FPGA ##
# Load the FPGA bitstream and place it in RAM
fatload mmc 0:1 $fpgadata soc_system.rbf;
fpga load 0 $fpgadata $filesize;
# Enable HPS-to-FPGA bridges
run bridge_enable_handoff;

echo ## Setting Env Variables and calling Linux ##
# Load device tree
setenv fdtimage soc_system.dtb;
# Locate root filesystem
setenv mmcroot /dev/mmcblk0p2;
# Command for loading kernel
setenv mmcload 'mmc rescan;${mmcloadcmd} mmc 0:${mmcloadpart} ${loadaddr} ${bootimage};${mmcloadcmd} mmc 0:${mmcloadpart} ${fdtaddr} ${fdtimage};';
# Command for booting kernel
setenv mmcboot 'setenv bootargs console=ttyS0,115200 root=${mmcroot} rw rootwait; bootz ${loadaddr} - ${fdtaddr}';

# Load kernel
run mmcload;
# Boot kernel
run mmcboot;
```
Now that we have a boot script, we need to compile it so that U-Boot can use it:
```bash
mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "Boot Script Name" -d boot.script u-boot.scr
```
That creates the compiled boot script file u-boot.scr, and with that, we are done with U-Boot!
 
 
# Generating and compiling the device tree
This guide will not into detail what the device tree is and what it is used for, because
the guide at Rocketboards.org does an excellent job explaining it. The SoC EDS tool that
we installed contains an application called sopc2dts that takes a .sopcinfo file and
optional .xml board files, and generates a .dts file, which is the source version of our
device tree. If we did not have this tool, then we would have to create a .dts file by
hand which can be very tedious. <br>
There are two .xml board files that are in the GHRD:
hps_common_board_info.xml and soc_system_board.info.xml. These files are for external
peripherals that you can connect to the HPS (ARM cores) part of the system, but since Qsys
does not know about peripherals that lie outside the FPGA fabric, they are described in
those two .xml board files. <br>
We now have to compile this .dts file to obtain a .dtb binary
device tree file. To do all of this, execute the following in the same terminal you have
been using up till now (if you did everything correctly, you should still be in the
DE10_NANO_SoC_GHRD/software/ folder before executing the following snippet):

```bash
sopc2dts --input soc_system.sopcinfo --output soc_system.dts --type dts --board soc_system_board_info.xml --board hps_common_board_info.xml --bridge-removal all --clocks
```
This generates the .dts file. Execute the following to compile it:
```bash
dtc -I dts -O dtb -o soc_system.dtb soc_system.dts
```

# Getting the Linux kernel source code

The following packages are needed for compiling the kernel:
```bash
sudo apt install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev libmpc-dev libgmp3-dev autoconf bc
```
Clone altera kernel for de10nano 
```bash
git clone https://github.com/altera-opensource/linux-socfpga
cd linux_socfpga
git checkout socfpga-5.4.54-lts
```
## Configuring the Linux kernel
Make sure that the CROSS_COMPILE
environment variable has been set and is pointing to the correct version of the GCC
toolchain, because we cannot configure nor compile the Linux kernel if is is not set. <br>
Later we need this compiler to write kernel module.<br>
First let’s create a default configuration:
```bash
make ARCH=arm socfpga_defconfig
```
We are now ready to configure the Linux kernel:
```bash
make ARCH=arm menuconfig
```
There are many options that you can modify and they lie outside the scope of this guide. For more information about an option, just enter “?” when hovering over that option. Although the default settings are pretty much ok, we still need to make some changes.

There are a few ways to flash your FPGA design. The traditional way is to just connect to the USB blaster interface and just flash it away. However, with the ARM HPS on our SoC, we have a couple of other ways as well:

Flash from U-Boot on boot - U-Boot has the ability to flash the FPGA design using some built-in commands. Flash from Linux while running - I think this is one of the big benefits of having an HPS.
We can flash our hardware design directly from Linux without even rebooting the device.

To be able to do this, we need to enable the Overlay filesystem support and Userspace-driven configuration filesystem in the kernel. If you don't intend to flash your FPGA from linux then feel free to skip these.

> NOTE - If you don't need these 2 options, you can just use the mainstream linux source to build your kernel instead of the altera-opensource version. Just clone the repository at github.com/torvalds/linux.

#### Kernel Options
``-> general``<br>
Uncheck **Automatically append version information to the version string** this makes it easier to test different versions of the drivers. Better to keep it enabled in production though.

``-> filesystems``<br>
Enable Overlay filesystem support
Under File systems, enable Overlay filesystem support and all the options under it.

``-> filesystems->pseudo filesystems``<br>
Enable CONFIGFS
This should be enabled already, but if not, do enable it:

## Compiling the kernel
To compile the kernel just execute the following:
```bash
make ARCH=arm LOCALVERSION=zImage -j 24
```
Once this is done, we will have a zImage file which is a compressed Linux kernel image.

# ROOTFS

https://blog.lazy-evaluation.net/posts/linux/debian-armhf-bootstrap.html
https://github.com/zangman/de10-nano/blob/master/docs/Debian-Root-File-System.md
https://github.com/zangman/de10-nano/blob/master/docs/%5BOptional%5D-Setting-up-Wifi.md#install-the-necessary-firmware

## Summary
There are several flavours of rootfs to choose from. This one focuses on the Debian
rootfs. This will give you an environment similar to the Rasbperry Pi OS for your
DE10-Nano. 

## Debootstrap and QEMU
Debootstrap is a utility that makes it easy to create a rootfs on an existing Debian based machine. However, since our host machine is most likely x86_64 and we are targeting an armhf architecture for the DE10-Nano, we will need to install an emulator from the QEMU project. To install both on your host OS:
```bash
sudo apt install debootstrap qemu-user-static
```
### First Stage
In the first stage, we will create a directory to hold the rootfs. Note that almost all the commands in this part will be done as root using sudo so take care not to make any mistakes.
```bash
mkdir rootfs
```
buster is the latest debian version at the time of writing.
Replace it with whatever is the latest.
```bash
sudo debootstrap --arch=armhf --foreign buster rootfs
```
### Second Stage
First, we have to copy over qemu to the target file system and chroot to target. Without copying over qemu we cannot chroot.

```bash
sudo cp /usr/bin/qemu-arm-static rootfs/usr/bin/
```
Now we should be able to chroot:
```bash
sudo chroot rootfs /usr/bin/qemu-arm-static /bin/bash -i
```
Once in the chroot-ed environment, we can kick off the second stage of debootstrap:
```bash
/debootstrap/debootstrap --second-stage
```
This will take about 5 minutes to complete.


#### Configuration
While still in the chroot environment, let's do some setup so that our rootfs is more convenient to use.

```bash
apt install vim -y
```
Hostname - Change the name to be different from the current host distro in /etc/hostname. I call mine de10-nano.

Root password - Set the root password so it is not blank. I just set it to 'root'.
```bash
passwd
```
fstab - Let's update fstab so that the system auto mounts the drives. Copy the following lines as is into /etc/fstab:
```bash
none		/tmp	tmpfs	defaults,noatime,mode=1777	0	0
/dev/mmcblk0p2	/	ext4	defaults	0	1
```
Enable the serial console - This allows you to see all the messages at boot time without having to ssh into the device using a simple serial console (Putty, minicom etc)
```bash
systemctl enable serial-getty@ttyS0.service
```
Locales - Configure and install the locales you will need. For me, this is just en_US.UTF-8:
```bash
apt install locales -y
dpkg-reconfigure locales
```
Ethernet - To get the ethernet on the DE10-Nano working, we need to add the following to the file/etc/network/interfaces under the line that says source-directory /etc/network/interfaces.d. This will enable DHCP:
```
auto lo eth0
iface lo inet loopback

allow-hotplug eth0
iface eth0 inet dhcp
```
Sources.list - Use a more complete apt sources.list. Edit the file /etc/apt/sources.list and add the following. Replace buster with whatever version of debian you are using:
```
deb http://deb.debian.org/debian/ buster main contrib non-free
deb-src http://deb.debian.org/debian/ buster main contrib non-free
deb http://deb.debian.org/debian/ buster-updates main contrib non-free
deb-src http://deb.debian.org/debian/ buster-updates main contrib non-free
deb http://deb.debian.org/debian-security/ buster/updates main contrib non-free
deb-src http://deb.debian.org/debian-security/ buster/updates main contrib non-free
```
Openssh-Server - Install openssh-server so that you can ssh into the device:
```bash
apt install openssh-server -y
```
(Optional) Root login over ssh - If you want to ssh as root, add/uncomment the following line in /etc/ssh/sshd_config:
```
PermitRootLogin yes
```

PRNG entropy seeding speedups - This speeds up the ssh server startup time on debian buster. See here for more details.
```bash
apt install haveged -y
```
Install any other packages - You can install any other packages you need as well:
```bash
apt install net-tools build-essential device-tree-compiler -y
```
Explanation:

``net-tools`` makes the ifconfig command available.
``build-essential`` install gcc and allows you to compile programs on the DE10-Nano.
``device-tree-compiler`` is needed to compile the device tree when flashing the FPGA directly from the HPS.

Clean up
Run the following for some basic clean up:
```bash
# Remove cached packages.
apt clean

# Remove QEMU.
rm /usr/bin/qemu-arm-static

# Exit from the chroot.
exit
```
Create a tarball
The last step is to create a tarball that we will extract into our SD Card partition:

```bash
cd rootfs

# Don't forget the dot at the end.
# Also has to be run as root.
sudo tar -cjpf $DEWD/rootfs.tar.bz2 .

# One level up to see the file.
cd ..
```
And that completes the Debian rootfs. Be careful when extracting this as it will extract everything within the same directory (without creating another directory).


# Create sdcard image
### Create the SD Card image file

Let's create a folder in which we'll create our SD card image:

```bash
 
mkdir sdcard
cd sdcard
```

Let's create an image file of 1GB in size. Adjust the `bs` parameter if you want a bigger or smaller one.

```bash
# Create an image file of 1GB in size.
sudo dd if=/dev/zero of=sdcard.img bs=950M count=1
```

> **Note**: For larger images, there are several more practical options available now instead of `dd` (such as `fallocate`). I refer you to [this stackoverflow question](https://stackoverflow.com/questions/257844/quickly-create-a-large-file-on-a-linux-system).
>
> For example, to create a 10GB image quickly `fallocate` is much faster:
>
> ```bash
> fallocate -l 10G sdcard.img
> ```

Let's make it visible as a device. This will enable us to treat this file as a disk drive so we can work with it.

```bash
sudo losetup --show -f sdcard.img
```

This should output the loopback device. You should see something like `/dev/loop0`.

### Partitioning the drive

To run Embedded Linux on the DE10-Nano, we need 3 partitions as shown in the table below. Partition numbers have to be exactly as shown below. File sizes also have to be exactly as shown, except for the Root Filesystem which can be increased to take up all the remaining space on the SD card. In our case, we're using a 1GB image file so, we'll have the Root Filesystem take up around 750MB.

Note that you should create them in the exact order listed when using fdisk.
> **Suggestion**: If you are creating an image and writing to an SD card several times because you are trying some experimental features, it's better to keep the file size low like 1GB for the entire SD card. This makes it faster to write the image to the SD Card.

| Order | Partition              | Partition Type | Partition Number | Last Sector | FS Type       | FS Hex Code |
| ----- | ---------------------- | -------------- | ---------------- | ----------- | ------------- | ----------- |
| 1     | U-Boot and SPL         | primary        | 3                | +1M         | Altera Custom | a2          |
| 2     | Kernel and Device Tree | primary        | 1                | +254M       | fat32         | b           |
| 3     | Root Filesystem        | primary        | 2                | _default_   | ext4          | 83          |

To partition the file, we're going to use `fdisk`:

```bash
sudo fdisk /dev/loop0
```

Press `p` and `enter` to see the list of partitions:

```bash
Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/loop0: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xef52db22

Command (m for help):
```

#### Bootloader partition

As you can see, there are no partitions at the moment. Let's create them as per the table above. For the first partition, you will need to type in the following commands in the fdisk prompt. For example, the first step is `n` followed by `enter`.

1. `n`, `enter`
2. `p`, `enter`
3. `3`, `enter`
4. `enter`
5. `+1M`, `enter`

If you typed everything correctly, it should look as shown below:

```bash
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 3
First sector (2048-2097151, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-2097151, default 2097151): +1M

Created a new partition 3 of type 'Linux' and of size 1 MiB.

Command (m for help):
```

You can see that it assigned it the default filesystem of `Linux`. We need to change that to `Altera Custom`. This is not a standard filesystem, so we'll need to manually assign the hex code `a2`. For this, enter the following commands:

1. `t`, `enter`
2. `a2`, `enter`

```bash
Command (m for help): t
Selected partition 3
Hex code (type L to list all codes): a2
Changed type of partition 'Linux' to 'unknown'.
```

#### Kernel and Device Tree partition

For the next partition, type the following commands:

1. `n`, `enter`
2. `p`, `enter`
3. `1`, `enter`
4. `enter`
5. `+254M`, `enter`

The output should look like:

```bash
Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1,2,4, default 1): 1
First sector (4096-2097151, default 4096):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4096-2097151, default 2097151): +254M

Created a new partition 1 of type 'Linux' and of size 254 MiB.

Command (m for help):
```

Again, let's change the filesystem type. Type the following commands:

1. `t`, `enter`
2. `1`,`enter`
3. `b`, `enter`

```bash
Command (m for help): t
Partition number (1,3, default 3): 1
Hex code (type L to list all codes): b

Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help):
```

#### Root Partition

For the last partition, we will assign whatever space remains in the image file. Here are the commands:

1. `n`, `enter`
2. `p`, `enter`
3. `2`, `enter`
4. `enter`
5. `enter`

```bash
Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2,4, default 2): 2
First sector (524288-2097151, default 524288):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (524288-2097151, default 2097151):

Created a new partition 2 of type 'Linux' and of size 768 MiB.
```

We will keep the default `Linux` partition type for this.

#### Writing the partition table

Check that the partitions are created as expected. Here is what I see when I type in `p`, `enter`:

```bash
Command (m for help): p
Disk /dev/loop0: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xef52db22

Device       Boot  Start     End Sectors  Size Id Type
/dev/loop0p1        4096  524287  520192  254M  b W95 FAT32
/dev/loop0p2      524288 2097151 1572864  768M 83 Linux
/dev/loop0p3        2048    4095    2048    1M a2 unknown
```

The partitions created so far haven't been written to the image file yet. So let's put them in with the command `w`, `enter`:

```bash
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Invalid argument

The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8).
```

The error in the message tells us that the partitions haven't been loaded by the kernel, which is indeed the case if you type:

```bash
ls /dev/loop0*
```

you will see we only have one device `/dev/loop0` and the partitions are not visible.

To access them for mounting and writing the contents, we have to run

```bash
sudo partprobe /dev/loop0
```

Now if you type:

```bash
ls /dev/loop0*
```

You should see the partitions:

```bash
/dev/loop0  /dev/loop0p1  /dev/loop0p2  /dev/loop0p3
```

### Creating the file systems

Lets create the fat and ext4 filesystems:

```bash
# Partition 1 is FAT
sudo mkfs -t vfat /dev/loop0p1

# Partition 2 is Linux
sudo mkfs.ext4 /dev/loop0p2
```

### Writing to the partitions

Now we'll populate the various partitions.

#### Bootloader partition

The bootloader partition is a binary partition which needs to be written in raw format. We don't have to mount it, so we'll just use the `dd` command to write it directly:

```bash
 
cd sdcard
sudo dd if=spl_bsp/preloader-mkpimage.bin of=/dev/loop0p3 bs=64k seek=0 oflag=sync
```

#### Kernel and Device Tree partition

This is a fat partition, so we'll need to mount it first and copy the files:

```bash
 
cd sdcard
mkdir -p temp_mnt

# Mount the temp_mnt partition.
sudo mount /dev/loop0p1 temp_mnt

# Copy the kernel image.
sudo cp ../linux-socfpga/arch/arm/boot/zImage temp_mnt

# Copy uboot, script
sudo cp ../u-boot-socfpga/u-boot.img ../u-boot.scr temp_mnt

# Copy rbf if you want fpga flashed during booting.
# Uboot expects this name: soc_system.rbf 
sudo cp ../../output_files/soc_system.rbf temp_mnt/

# Copy the de0 device tree.
sudo cp ../soc_system.dtb temp_mnt

# Unmount the partition.
sudo umount temp_mnt
```

#### Root Filesystem partition

For the final partition, here are the steps:

```bash

# Mount to temp_mnt
sudo mount /dev/loop0p2 temp_mnt

# Extract the rootfs archive.
cd temp_mnt
sudo tar -xf ../../rootfs.tar.bz2 .

# Unmount the partition.
cd ..
sudo umount temp_mnt
```

Quite straightfoward.

### Cleanup

Run the following commands to clean up:

```bash
# Delete unnecessary files and folders.
cd sdcard
rmdir temp_mnt


# Delete the loopback device.
sudo losetup -d /dev/loop0
```

### Writing to SD Card

The hard work is done. Now all that's left is to write the `sdcard.img` file to an actual SD Card and we're ready to boot.

#### Writing in Linux

If you are not using Virtualbox for your Debian OS or if you have access to the SD Card device directly in the virtual machine, then you can use the following command to write the image file directly to the SD Card.

> **WARNING** - Be extremely careful with the command below. If you type the wrong device, there are no warnings, it will wipe your system clean.

```bash
 
cd sdcard

# Identify your SD Card device.
lsblk

# Write to the correct device (Ex: /dev/sdb).
sudo dd if=sdcard.img of=/dev/sdb bs=1M status=progress
```

##### Note: Writing to a used SD Card

This is taken from [this stack exchange link](https://raspberrypi.stackexchange.com/a/108628).

If we are writing to an SD card that was already written to with an image file before, we need to delete all the partitions on the sd card before writing to it again.

To do this, we can use fdisk as follows:

1.  `d`, `enter`
1.  `enter`
1.  `d`, `enter`
1.  `enter`
1.  `d`, `enter`
1.  `enter`
1.  `w`, `enter`

Example output below:

```bash
Command (m for help): d
Partition number (1-3, default 3):

Partition 3 has been deleted.

Command (m for help): d
Partition number (1,2, default 2):

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): d
No partition is defined yet!

Command (m for help): w

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

After this, we can run the `dd` command as shown above.

#### Writing in Windows

If, like me, your SD Card reader is attached to your laptop and not available as a USB device to access through virtualbox, you will need to transfer the file to windows and then use [Rufus](https://rufus.ie/) to write to the SD Card.

You can transfer the file either by creating a shared folder to transfer between virtualbox and the host windows system or by uploading it to Google Drive and then downloading in windows.


### Note on updating sdcard.img

I wasted a lot of time on this, so just putting it here in case you run into the same.

When experimenting with different builds of U-Boot, I was reusing the same `sdcard.img` file and simply running `dd` to overwrite the binary partition, which is `/dev/loop0p3` in our guide. However, when I did this and wrote to the SD Card it kept using the old version of the bootloader. After a lot of time debugging, it turns out the reason it wasn't updating was because `dd` doesn't wipe the entire partition. It just streams whatever you have to the destination. So the bits may get overwritten or maybe not.

The solution is to wipe the partition clean before updating it. This can be done with the following command which fills it with zeros till the partition runs out of space:

```bash
sudo dd if=/dev/zero of=/dev/loop0p3 bs=64k oflag=sync status=progress
```
