---
title: "Installing Debian Linux on a Surface 3 with Bad eMMC"
date: 2025-08-08
---

## The Problem

Some time ago I obtained a Surface 3 tablet second-hand. The screen is small
and the keyboard cramped and it only has 2GB of RAM, but it's OK for light
usage when I'm traveling or just feel like sitting outside. Not long after this
the eMMC inside died (a common problem it seems), fortunately taking nothing
important with it.

Can I replace the EMMC? Technically possible, but it's soldered in and I don't
want my tablet ending up like [this one](https://www.ifixit.com/Guide/Microsoft+Surface+3+Battery+Replacement/52684).

This thing has a readily-accessible SD slot, [but I can't boot from it](https://superuser.com/questions/1884927/can-i-use-an-sd-card-as-a-boot-drive-on-the-surface-3-if-it-has-no-sd-card-boot).

I could install to a USB drive (those can be pretty small these days), but
there's only one USB port and I don't want to tie it up forever or leave a
dongle hanging out of it.

So that leaves pretty much one option (besides buying [something new](https://pine64.org/devices/pinetab2/)):
Use a USB drive as a boot device and install the OS to the SD card! Not an
ideal solution maybe, but there are no ideal solutions.

## The Process
The following was done with Debian 12.11.0.

In order to get this to work, we need to do a few things:
  1. Prepare the boot drive
  2. Disable Secure Boot (not strictly necessary but getting it to work is
     annoying)
  3. Install Debian to the SD card/USB drive
  4. Modify the installation to better support our setup

Required material:
  * A boot USB drive, 1GB minimum recommended
  * A Debian installation USB drive
  * An SD card for the OS, 8GB bare minimum though 16GB is more realistic

Recommended material:
  * A USB hub, preferably with an ethernet port but the installation image
    contains the correct wifi drivers so that's not required any more

#### Prepare the Boot Drive
In order to work correctly, the boot drive needs two partitions:
  * An EFI system partition
  * A boot partition

The drive can be partitioned in either GPT or MBR format without issue. This
can be performed by the Debian installer if the USB drive is accessible.

The EFI system partition type must be set correctly - `uefi` is the short name
if using `fdisk`. This partition doesn't need to be very big at all (I've used
64MB but fewer than 6MB is actually used; the Arch wiki recommends [at least
512MB](https://wiki.archlinux.org/title/EFI_system_partition#Create_the_partition)
which has the added bonus of allowing that partition to be used for general
files storage). It must be formatted as FAT32.

The boot partition can be any type supported by GRUB. The most reliable is
probably ext2. This partition needs to hold all the initrds and kernels
installed on the system which can add up over time - there will be at least
two at the end of these instructions (one installed by default plus the
linux-surface kernel) so 512MB is probably as low as you want to go but 1GB
is better.

#### Disable Secure Boot
This one is pretty easy - with the device off, hold `volume up` then press the
`power` button and wait for it to boot into the UEFI menu. Secure Boot can be
disabled there.

The boot order may also need to be changed so that the USB drive boots before
the SSD.

#### Install Debian
The process here is pretty straightforward - follow the [installation instructions](https://www.debian.org/releases/stable/amd64/index).

When it comes time to partition, select manual partitioning.

###### USB Hub Method
Select your boot partition as `/boot` and the EFI partition created before as
the `EFI System Partition`.

Complete the installation as normal, and skip to [Finish the Installation](#finish-the-installation)
after rebooting.

###### No USB Hub Method
For this method you'll need either another computer that can read both the SD
card and boot drive or a liveusb that can be loaded into RAM like [SystemRescueCD](https://www.system-rescue.org/).

When partitioning the SD card, you'll need an EFI partition set up like the
USB drive above. If you create a separate boot partition on the SD card with
the same file system as the USB boot partition some things are simplified later
on but it doesn't make a very big difference.

Install the system to the SD card as you normally would. When done, either boot
up a liveusb and then swap in the boot USB drive or insert the boot USB drive
and SD card into another computer and make note of their device names.

The following commands are all performed as `root` or with `sudo`. Your device
names and partition numbers may differ.

Mount the root filesystem and the boot drive:
```
mkdir /tmp/sf3_root
mount /dev/mmcblk0p2 /tmp/sf3_root`

# Only do this if /boot is a separate partition:
mount /dev/mmcblk0p3 /tmp/sf3_root/boot`

mkdir /tmp/sf3_root/usb_boot
mount /dev/sda2 /tmp/sf3_root/usb_boot
```
And the required system directories:
```
for d in dev sys proc; do mount --rbind /$d /tmp/sf3_root/$d; mount --make-rslave /tmp/sf3_root/$d; done
```
The `mount --make-rslave` commands will allow us to unmount the partitions later
without having to deal with any `umount: /tmp/sf3_root/dev/shm: target is busy.`
error messages.

Enter the chroot:
```
chroot /tmp/sf3_root /bin/bash -l
```
And then copy the required boot files from `/boot` to `/usb_boot`:
```
cp -ai /boot/{System*,config*,vmlinuz*,initrd*} /usb_boot/
mkdir /usb_boot/grub
cp -ai /boot/grub/grub.cfg /usb_boot/grub/grub.cfg
```
And remount our boot drive in it's final location and mount the EFI partition:
```
umount /usb_boot
# Only do this if /boot is a separate partition:
umount /boot
mount /dev/sda2 /boot
mkdir /boot/efi
mount /dev/sda1 /boot/efi
```
Now we need to install GRUB:
```
grub-install --target=x86_64-efi --boot-directory=/boot --efi-directory=/boot/efi --recheck --removable
```
If `/boot` *was not* a separate partition, the file `/boot/grub/grub.cfg` contains
references to the original boot folder layout which need to be manually changed.
Open the file for editing (which requires root permissions) and look for lines
of the form:
```
linux	/boot/vmlinuz-6.1.0-37-amd64 root=UUID=ffa49137-324a-452f-9bff-f1d0d5520b3e ro  quiet
initrd	/boot/initrd.img-6.1.0-37-amd64
```
and change them to instead read:
```
linux	/vmlinuz-6.1.0-37-amd64 root=UUID=ffa49137-324a-452f-9bff-f1d0d5520b3e ro  quiet
initrd	/initrd.img-6.1.0-37-amd64
```
then add the `/boot` entry to `/etc/fstab`:
```
# If the USB boot partition is labeled:
LABEL=your_boot_label /boot ext2 defaults,relatime,lazytime 0 0
# Otherwise:
UUID=the_partition_uuid /boot ext2 defaults,relatime,lazytime 0 0

```

If `/boot` *was* a separate partition, edit the device UUID in `/etc/fstab` to
match the new device (run `ls -l /dev/disk/by-uuid` and look for the link to
the partition). If you gave the USB boot partition a label when partitioning
you can replace `UUID=...` with `LABEL=your_boot_label`.

Finally, exit the chroot and unmount everything:
```
exit
umount -R /tmp/sf3_root/{dev,sys,proc}
umount /tmp/sf3_root/boot/efi
umount /tmp/sf3_root/boot
umount /tmp/sf3_root

```
You can now boot into the OS. GRUB may issue a warning about not being able to
find a device and require you to press a key to continue but it should work fine
after that. If it doesn't you may need to edit `grub.cfg` and change any GRUB
commands that set the root to point to the new `/boot`.

Once booted, run `update-grub` to regenerate `/boot/grub/grub.cfg` to remove the
GRUB error.

#### Finish the Installation
To be really done, we need to:
  1. Prevent `/boot` from automounting so we can remove the USB drive
  2. Install the linux-surface kernel for better hardware support

###### Prevent /boot from Automounting
Edit the `/boot` and `/boot/efi` entries in `/etc/fstab` to add the `noauto` option:
```
# We had this:
#UUID=... /boot     ext2 defaults,relatime 0 2
#UUID=... /boot/efi vfat defaults,relatime 0 1

# We want this:
UUID=... /boot     ext2 defaults,relatime,noauto 0 2
UUID=... /boot/efi vfat defaults,relatime,noauto 0 1
```
One hiccup with this is that `/boot` needs to be mounted whenever `/boot/grub/grub.cfg`
or the initrds are re-generated (e.g. with kernel updates). If you forget to
do this and the old kernel is still installed on the system then the only problem
will be that you'll boot into the old kernel, but if you try to boot into a
system with a kernel that's been removed then you won't have access to any
modules - you'll be limited to whatever is built in or that the initrd loaded.
To generate the files at a later time, just mount `/boot` and run:
```
update-initramfs -c -k <latest-kernel-version>
update-grub
```

###### Install the linux-surface kernel
The [linux-surface kernel](https://github.com/linux-surface/linux-surface) is
a branch of the Linux kernel with additional drivers and fixes for Surface
devices. Instructions for installation can be found [here](https://github.com/linux-surface/linux-surface/wiki/Installation-and-Setup#Debian--Ubuntu).

Reboot into the new kernel and you're done.
