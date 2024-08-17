---
title: "Replacing the drive in a single drive Proxmox node"
date: "2024-01-14"
description: "A (non-exhaustive) guide to replacing the boot drive in a Proxmox node"
categories:
  - "homelab"
  - "proxmox"
tags:
  - "proxmox"
  - "homelab"
  - "sysadmin"
---

Recently, I started getting the dreaded emails from `smartd` on one of my Proxmox nodes. The drive was (slowly) failing and needed to be replaced. Unfortunately due to my nodes being [micro form-factor(MFF)](https://www.dell.com/support/kbdoc/en-uk/000125345/optiplex-3070-visual-guide-to-your-computers#MFF_Front), they only have a single drive. This means that I would also need to migrate the VM disks to the new drive too. As this is a homelab setup, I don't have a ton of capacity or any fancy storage solutions, so I would need to get a little creative and replace the drive in-situ.

The 'official' method here would be to back everything up to Proxmox Backup Server, install a fresh copy of Proxmox onto the new drive and then restore from the backups, however setting up the backup server is something which has been on my todo list for a while (yes, I know).

So on the face of it, the rough steps I had in my head formed a relatively simple process which I've done dozens of times in my previous day job as a sysadmin, however I did discover a few potential pitfalls along the way.

## My setup

Each node has a single drive and I use the `local-lvm` (LVM thin volumes) storage type for the VM disks. Each node boots in UEFI mode (which will be important later) with GRUB as the bootloader.

## High-level overview

There aren't many steps to this process:

1. Duplicate the partition table to the new drive.
2. Migrate the VM disks to the new drive.
3. Migrate the root OS over.
4. ??
5. Profit.

Steps 1-2 can be completed whilst booted into Proxmox (VMs must be shut down though), however the remaining steps will require a live environment which can be booted from a USB stick.

## Duplicating the partition table

In my case, I was replacing the drive with one of an identical size, however if you choose to use a different sized drive then the following steps will need to be adjusted accordingly. As noted earlier, because my nodes are micro form-factor, I don't have an internal slot for a second drive so I needed to connect the new drive via an m.2 NVME to USB enclosure.

In the following code snippets, my old drive is `nvme0n1` and the new drive is `sda`.

Wih the groundwork laid and both drives connected, you can use `sgdisk` to duplicate the partition table:

```shell
sfdisk -d /dev/nvme0n1 | sfdisk /dev/sda
```

And we then need to randomise the GUIDs so that there are no collisions between the two drives:

```shell
sgdisk -G /dev/sda
```

Finally, make sure the kernel is aware of the new partition table:

```shell
blockdev --rereadpt /dev/sda
```

## Migrating the VM disks

My initial plan had been to add the new drive as a physical volume to the existing `pve` volume group and then use `pvmove` to migrate all the logical volumes over. However, I discovered that `pvmove` doesn't work with LVM thin volumes (my VM disks are all thin-provisioned) so I was forced to come up with a different plan.

The method I arrived at was to create an entirely new volume group and then mirror the logical volumes (size, configuration etc). Once the layout looked the same, I could use `dd` to copy the VM filesystems to the new logical volumes.

My existing drive has three partitions: a `bios_grub` partition (`sda1`), an ESP partition (`sda2`) and the main LVM partition (`sda3`). At this point, we are only going to be working with the LVM partition.

First, we create the new physical volume and then use it to create a volume group called `new`:

```shell
pvcreate /dev/sda3
vgcreate new /dev/sda3
```

Now we can begin replicating the logical volumes on the new drive. It's important to get the sizes and names correct here; sizes can be obtained using `lvdisplay` against the existing volumes:

```shell
$ lvdisplay pve/root
  --- Logical volume ---
  LV Path                /dev/pve/root
  LV Name                root
  VG Name                pve
  LV UUID                VBvGFf-Ham2-aN5u-wPmA-zgz3-hm61-x4b3Kw
  LV Write Access        read/write
  LV Creation host, time proxmox, 2021-04-01 15:58:16 +0100
  LV Status              available
  # open                 1
  LV Size                50.00 GiB
  Current LE             12800
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1
```

Using the value of `LV Size` we can first create the OS's volumes (adjust the sizes accordingly for your setup):

```shell
lvcreate -L 50G -n root new
lvcreate -L 4G -n swap new
```

Next, we need to create the thin pool (named `data`) which will hold the VM disks:

```shell
lvcreate --type thin-pool -L 164.61G -n data new
```

And with this in place, we can begin adding volumes for each virtual machine:

```shell
lvcreate -T new/data -L 15G -n vm-201-disk-0
```

Special consideration needs to be given to any template images which are used to create new VMs as these require different flags to be set at creation time:

```shell
lvcreate -V 5G -T new/data --permission r --setactivationskip y -n base-123-disk-0
```

`--permission r` sets the volume to read-only and `--setactivationskip y` prevents the volume from being activated when the machine boots.

## Cloning VM filesystems

With the new logical volumes created, we can now use `dd` to clone the filesystems over. It's important to use the `sparse` flag here to ensure that only the used blocks are copied as these are thin-provisioned volumes:

```shell
dd if=/dev/pve/vm-401-disk-0 of=/dev/new/vm-401-disk-0 bs=4194304 conv=sparse status=progress
```

And obviously ensure that the source (`if`) and destination (`of`) volumes are correct.

As before, extra consideration needs to be given to the template images. They must first be activated manually and then also set to read-write before using `dd`:

```shell
lvchange -a y new/base-123-disk-0 -K
lvchange new/base-123-disk-0 -p rw
```

Use the same `dd` command as before, and then revert the volume settings:

```shell
lvchange -a n new/base-123-disk-0
lvchange new/base-123-disk-0 -p r
```

With all VM disks successfully cloned, it's time to boot into a live environment and migrate the root OS. Any distro will suffice as long as it has LVM tools available.

## Migrating the root OS

Once inside the live environment, `dd` can be used again to clone the volume. As the root volume isn't thin-provisioned, the flags differ slightly:

```shell
dd if=/dev/pve/root of=/dev/new/root bs=4194304 status=progress conv=fdatasync
```

Finally, we need to make the new drive bootable. First, let's mount up the new root volume and chroot into it:

```shell
mount /dev/new/root /mnt
mount --bind /dev/ /mnt/dev/
mount --bind /sys/ /mnt/sys/
mount /dev/sda2 /mnt/boot/efi
chroot /mnt
mount /proc
```

Then we need to install GRUB to the new drive:

```shell
grub-install --recheck/dev/sda
```

And also update the GRUB configuration:

```shell
update-grub
```

For good measure, it's also worth regenerating the initramfs(es):

```shell
update-initramfs -kall u
```

## Populating the ESP partition

The final step in making the new drive bootable is to populate the ESP partition with the required files. Fortunately, Proxmox provides a tool called [`proxmox-boot-tool`](https://pve.proxmox.com/wiki/Host_Bootloader#sysboot_proxmox_boot_tool) which makes this pretty painless.

Note: the following commands should be run from inside the chroot environment you created previously.

First, format the ESP partition:

```shell
proxmox-boot-tool format /dev/sda2
```

And then initialise the partition (omit `grub` if you are using `systemd-boot`):

```shell
proxmox-boot-tool init /dev/sda2 grub
```

Finally, populate the ESP partition:

```shell
proxmox-boot-tool refresh
```

You can check that the system is configured correctly using the `status` subcommand:

```shell
$ proxmox-boot-tool status
Re-executing '/usr/sbin/proxmox-boot-tool' in new private mount namespace..
System currently booted with uefi
E47D-4BCC is configured with: grub (versions: 6.5.13-5-pve, 6.8.8-2-pve)
```

## Rename volume groups

The last step before rebooting is to rename the volume groups so that the new drive looks right as far as Proxmox is concerned. First, we need to exit the chroot (`ctrl+d`) and then unmount everything:

```shell
ctrl+d # exits the chroot
umount /mnt/boot/efi
umount /mnt/{proc,sys,dev}
umount /mnt
```

The we rename the volume group on the failing drive:

```shell
vgrename pve old
```

And then swap the new volume group into place:

```shell
vgrename new pve
```

## Finally

With everything complete, it's time to shut down the machine in order to swap your new drive in. Once the new drive is in place, boot the machine and cross your fingers.

## Wrapping up

Hopefully you find this helpful; it's as much a blog post as it is a reference for my future self.
