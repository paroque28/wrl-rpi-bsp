Raspberry Pi BSP
================

1. Build Instructions
---------------------

Use the following steps to configure a platform project for this BSP with
Wind River Linux LTS 17:

    $ git clone --branch WRLINUX_10_17_LTS /path/to/wrlinux-x
    $ ./wrlinux-x/setup.sh --distro wrlinux --dl-layers --accept-eula yes
    $ . environment-setup-x86_64-wrlinuxsdk-linux
    $ . oe-init-build-env
    $ bitbake-layers add-layer /path/to/wrl-rpi-bsp/

If you will build an SDK and plan to use it to build kernel modules then also
add `--templates feature/kernel-dev` when running `setup.sh` above.

Set `MACHINE` to the required board variant, one of:

  * rpi
  * rpi2
  * rpi3
  * rpi3-64 (64-bit)

for example:

    $ echo "MACHINE = \"rpi\"" >> conf/local.conf

for the original Raspberry Pi.

Build the platform with:

    $ bitbake wrlinux-image-glibc-std

and finally, if required, build and install the SDK:

    $ bitbake -c populate_sdk wrlinux-image-glibc-std
    $ cd tmp-glibc/deploy/sdk/
    $ ./wrlinux-10.17.41.1-glibc-x86_64-rpi-wrlinux-image-glibc-std-sdk.sh

See the User Guide for SDK usage instructions.


2. Boot Instructions
--------------------

### 2.1. SD Card Boot

The original Raspberry Pi has an SD card reader which can be used to boot the
device. Newer models use a microSD instead. The following instructions are
suitable for both.

#### 2.1.1. Partition and Format SD Card

    $ sudo fdisk /dev/mmcblk0

    Welcome to fdisk (util-linux 2.27.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    Command (m for help): n
    Partition type
       p   primary (0 primary, 0 extended, 4 free)
       e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (1-4, default 1):
    First sector (2048-3862527, default 2048):
    Last sector, +sectors or +size{K,M,G,T,P} (2048-3862527, default 3862527): +64M

    Created a new partition 1 of type 'Linux' and of size 64 MiB.

    Command (m for help): t
    Selected partition 1
    Partition type (type L to list all types): c
    Changed type of partition 'Linux' to 'W95 FAT32 (LBA)'.

    Command (m for help): n
    Partition type
       p   primary (1 primary, 0 extended, 3 free)
       e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (2-4, default 2):
    First sector (133120-3862527, default 133120):
    Last sector, +sectors or +size{K,M,G,T,P} (133120-3862527, default 3862527):

    Created a new partition 2 of type 'Linux' and of size 1.8 GiB.

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Synching disks.

    $ sudo mkfs.vfat -c -n BOOT /dev/mmcblk0p1
    mkfs.fat 3.0.28 (2015-05-16)

    $ sudo mkfs.ext4 -c -L ROOTFS /dev/mmcblk0p2
    mke2fs 1.42.13 (17-May-2015)
    Discarding device blocks: done
    Creating filesystem with 466176 4k blocks and 116640 inodes
    Filesystem UUID: 6ec92418-b8c6-4b19-b72e-1eb633023128
    Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

    Checking for bad blocks (read-only test): done
    Allocating group tables: done
    Writing inode tables: done
    Creating journal (8192 blocks): done
    Writing superblocks and filesystem accounting information: done

#### 2.1.2. Install Bootloader and Kernel

    $ sudo mount -t vfat /dev/mmcblk0p1 /mnt
    $ cd /mnt
    $ sudo cp -r <nonfreeLayerDir>/bootloader/broadcom/* .
    $ sudo cp <prjDir>/build/tmp-glibc/deploy/images/rpi/Image kernel.img
    $ cd
    $ sudo umount /mnt

**NOTE**:
  * The bootloader can be found in the wrl-rpi-nonfree layer.
  * The rpi2 and rpi3 kernel image file should be called `kernel7.img`.
  * The rpi3-64 kernel image file should be called `kernel8.img`.
  * For the rpi3 and rpi3-64, enable the micro-UART in config.txt if a serial
    console is required.
  * For the rpi3, change root device to `mmcblk1p2` in the cmdline.txt file.
  * For the rpi3-64, replace the `bcm2710-rpi-3-b.dtb` file with the version
    from the 64-bit subdirectory.

#### 2.1.3. Install rootfs

    $ sudo mount -t ext4 /dev/mmcblk0p2 /mnt
    $ cd /mnt
    $ sudo tar jxpf <prjDir>/build/tmp-glibc/deploy/images/rpi/wrlinux-image-glibc-std-rpi.tar.bz2

Optionally, enable mount of the boot partition on /boot of the rootfs:

    $ sudo rm boot/*
    $ sudo vi etc/fstab

        /dev/mmcblk0p1  /boot  vfat  defaults,sync  0  0

**Note**: Use `mmcblk1p1` for the rpi3.

Finally, unmount the card:

    $ cd
    $ sudo umount /mnt

The SD / microSD card is now ready for use.
