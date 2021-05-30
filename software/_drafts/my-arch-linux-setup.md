---
---

I've been a big fan of [Arch Linux] for a long time now. Though it is a pain to
set up, it offers a great opportunity to learn exactly how everything fits
together in a Linux system. There's also that pleasant satisfaction when
everything is finally set up exactly the way I like it. In the interest of
making this process easier, I've written down my personal guide to setting up
Arch Linux. Though this guide does set up Arch as a [VirtualBox] guest, a lot
of this applies to my other setups as well.

[Arch Linux]: https://archlinux.org/
[VirtualBox]: https://www.virtualbox.org/

* TOC
{:toc}

## Overview

Rather than just going through all the steps, I will focus on explaining the
reasoning behind my decisions. After all, the benefit of Arch is that you get
to make the choice, and I would hate to completely take that away from you.

Key choices I made:

| Boot loader          | [systemd-boot]     |
| Full disk encryption | [dm-crypt]         |
| File system          | [btrfs]            |
| Window manager       | [i3]               |
| Swap                 | [Swap file]        |
| Hibernation          | Not supported      |
| Networking           | [systemd-networkd] |
| Firewall             | TODO               |

[systemd-boot]: https://www.freedesktop.org/wiki/Software/systemd/systemd-boot/
[dm-crypt]: https://en.wikipedia.org/wiki/Dm-crypt
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[i3]: https://i3wm.org/
[Swap file]: https://wiki.archlinux.org/title/Swap#Swap_file
[systemd-networkd]: https://www.freedesktop.org/software/systemd/man/systemd-networkd.service.html

Obviously this guide is based on the [installation guide] and in fact should be
followed in conjunction with that guide since I don't plan on covering every
detail here. In general, the [ArchWiki] is an excellent resource and honestly
one of the strongest strengths of Arch Linux. It's even useful when you aren't
running Arch.

[installation guide]: https://wiki.archlinux.org/title/Installation_guide
[ArchWiki]: https://wiki.archlinux.org/

## Pre-installation Setup

To start off, download an installation image and then create a new VM in
VirtualBox. There are a variety of settings you can configure as you like, but
there are a few recommendations I would make.

First, enable EFI mode. Though it's perfectly fine to stick to BIOS mode, most
modern computers are using UEFI, so you might as well get used to it. If you do
decide to stick with BIOS mode, you'll need to choose a different boot loader
than [systemd-boot].

Second, I recommend choosing the disk size wisely. Though it's possible to
resize things later, it's more convenient to ensure you have enough space from
the beginning. Arch can be run very lightweight, but in my experience, 8 GB is
as low as you'd want to go and still have an acceptable desktop environment.
Even then, space will be tight if you start installing a lot of larger packages
(LaTeX, fonts, etc.) and that's not even including your own files. I personally
go with 32 GB so I don't need to worry about it.

Set up the VM to boot from the installation image and then boot it. This will
bring you into a live installation of Arch Linux that you will use to set up
your actual installation. Start by confirming you are running in UEFI mode and
have internet access. Then ensure the system clock is synced via NTP.

### Disk Partitioning

There are many ways to partition the disk for a Linux install depending on your
needs. This guide will aim to keep things simple, but before we start, there
are a few things to keep in mind.

First, since we are booting in UEFI mode, using [GPT] is recommended over [MBR]
since it is more flexible and better supported by motherboards and operating
systems implementing UEFI[^1]. Second, UEFI requires an [EFI system partition]
since that is where UEFI will look for boot loaders to run. Given that we are
trying to keep it simple, we will only have 2 partitions:

1. EFI system partition
1. Encrypted root directory partition (i.e. everything underneath `/`)

[GPT]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[MBR]: https://en.wikipedia.org/wiki/Master_boot_record
[^1]: The ArchWiki's entry on [partitioning](https://wiki.archlinux.org/title/Partitioning#Choosing_between_GPT_and_MBR) describes these reasons in more detail, but in general, there's little reason to use MBR unless you are stuck with legacy hardware.
[EFI system partition]: https://en.wikipedia.org/wiki/EFI_system_partition

We will not need a swap partition because we will be using a [swap file]
instead. Swap files are convenient to resize (or even delete entirely to free
up a bit of space). However, if you choose to run [btrfs] and wish to add more
disks in the future, you may want to use a swap partition since btrfs doesn't
support swap files on filesystems with multiple disks. On a VM, increasing the
disk's size isn't that hard, but on a real machine, just adding another disk is
way more convenient.

Though the OS can be in a separate partition, we will keep the kernel image in
the EFI system partition since we will not be encrypting our kernel image. In
theory, encrypting the kernel will prevent an attacker with physical access
from modifying the kernel, but in reality this won't stop them since they can
just modify the boot loader instead[^2]. With this in mind, the EFI system
partition is a convenient alternative, but we could've created another
partition just for the kernel. Note that if you are installing Arch on a system
that already has an EFI system partition, the existing partition may be too
small and you may be forced to put the kernel on a separate partition.

[^2]: This attack vector is known as the [Evil Maid Attack]. The correct way to prevent this is to use [Secure Boot], but in my opinion, that is overkill unless you worried about an intelligence agency coming for you.
[Evil Maid Attack]: https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html
[Secure Boot]: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Secure_boot

There are many tools that can be used to actually partition the drive so choose
whatever suits you (I personally just use `fdisk`). The ArchWiki recommends
making the EFI system partition 260 MB which I believe comes from the
[minimum FAT32 partition size on certain drives](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#system-partition).
Use the remainder of the disk as the second partition which will be storing the
root directory.

### Set Up Encryption

We will be using [dm-crypt] to encrypt the entire root directory partition. The
main reason to do this is to prevent someone from accessing your data if your
hard drive is stolen. However, you will need to type in the decryption password
every time you boot the system. For convenience, I usually make the decryption
password the same as my login password which is secure enough for me (though
note that nothing will keep the passwords in sync).

Following
[these instructions](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition),
you should first encrypt the partition via the `cryptsetup` command and set it
up to be mounted at `/dev/mapper/cryptroot`. At this point,
`/dev/mapper/cryptroot` represents the unencrypted drive so anything you write
there will be encrypted. Now, there are still more steps necessary to set up
automatic decryption of the partition at boot time, but we will get to that
later when we set up initramfs.

### Format Partitions

Next, you will need to format each partition with the correct file system. The
EFI system partition must be FAT32 (e.g. `mkfs.fat -F32 /dev/sda1`), but your
root directory partition can be whatever you like. If you encrypted your root
directory partition, ensure that you are formatting the unencrypted mount point
(e.g. `/dev/mapper/cryptroot`) rather than the partition itself otherwise you
will destroy the encryption. In this guide, we will be using [btrfs] since it
supports snapshots and automatic compression.

Now before we get any further, I know that btrfs has a pretty bad reputation
for losing data which is pretty much the worst possible thing a file system
could do. This is especially ironic since it is a [copy-on-write] file system
which are traditionally considered to be safer from corruptions since they
easily support atomic operations due to simply never overwriting previous
data/metadata. I believe most of these fears were due to people using btrfs
with a RAID5/6 setup which
[still isn't supported by btrfs](https://btrfs.wiki.kernel.org/index.php/RAID56).
Even ignoring that, there are still plenty of horror stories. I don't have much
experience with btrfs, but it has a lot of nice features and enough people
saying good things about it (it's now the
[default for Fedora](https://fedoraproject.org/wiki/Changes/BtrfsByDefault))
for me to justify using it. Still, if its reputation scares you off, feel free
to use a different file system.

[copy-on-write]: https://en.wikipedia.org/wiki/Copy-on-write

### Mounting Filesystems

Next, we need to decide how our newly formatted partitions will be mounted. If
you didn't choose to use [btrfs], the straightforward scheme is to just mount
your new root directory at `/mnt` and then mount the EFI system partition at
`/mnt/boot` (if you don't want to put your kernel image in the EFI system
partition, then do `/mnt/efi` instead).

However, with btrfs, we have some additional options. First, you should decide
whether to mount the btrfs directory with
[compression](https://btrfs.wiki.kernel.org/index.php/Compression)
enabled. I don't have any hard data on this, but for a hard drive, enabling
compression is pretty much a no-brainer since disk IO is slow enough to make it
worth the CPU cost to reduce it. For SSDs though, it's probably a perfomance
hit most of the time. Still, if disk space is more important to you, the
[perfomance penalty](https://git.kernel.org/pub/scm/linux/kernel/git/mason/linux-btrfs.git/commit/?h=next&id=5c1aab1dd5445ed8bdcdbb575abc1b0d7ee5b2e7)
is likely worth it (e.g. our VM with only 32 GB of space). If perfomance is
more important, you could choose the LZO algorithm since it is the fastest, but
at that point, you should probably just not enable compression at all. If you
do decide to enable compression, you should mount with the option
`compress=zstd` since the ZSTD algorithm seems better than the default ZLIB.

Next, you should mount with the [`noatime`] option. In general, this is a
useful optimization for any file system when you don't need access times for
your files, but this is especially true for btrfs since metadata updates are
implemented via copying which makes updating the access time even more
expensive.

[`noatime`]: https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs(5)#NOTES_ON_GENERIC_MOUNT_OPTIONS

#### Subvolume Layout

Finally, before we start mounting things, we should figure out how we want to
lay out our [subvolumes]. Subvolumes act as directory-like mount points that
maintain their own file tree even though they are all part of the same file
system. Most usefully for us, subvolumes support snapshotting. In btrfs, a
[snapshot] is just a subvolume that starts off the same as the file tree of
another subvolume. Once the snapshot is created, its file tree is essentially
independent of the original subvolume e.g. modifications to one will not affect
the other. Cheap snapshotting is one of the main advantages of a
[copy-on-write] file system.

[subvolumes]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes
[snapshot]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots

To start off, there are a
[few basic approaches](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout)
to layout your btrfs subvolumes. Feel free to come up with your own, but here's
the layout of my subvolumes and how they are mounted into the system.
Subvolumes are prefixed with `@` and normal directories are suffixed with `/`.

```
toplevel       -> /.btrfs
├── @home      -> /home
├── @root      -> /
├── snapshots/
│   ├── home/
│   │   ├── @2020-02-20T20:20:20
│   │   ├── @2021-02-21T21:21:21
│   │   └── @my_snapshot
│   └── root/
│       ├── @2002-02-02T02:02:02
│       ├── @2003-03-03T03:03:03
│       └── @before_install_foo
└── @swap
```

The `/.btrfs` mount point exists to make it convenient to access all the other
subvolumes/snapshots. `@home` and `@root` are separate subvolumes since most of
the time you'd want to restore them separately from each other (e.g. I deleted
my documents vs I need to downgrade a package). The `swap` subvolume is for
storing a [swap file] since it requires that the containing subvolume is not
snapshotted or compressed.

To actually create this layout, you should first mount the btrfs top-level
somewhere (e.g. `/btrfs`) so you can `cd` in and start creating the subvolume
and directory structure. To continue with the rest of the installation, you
should mount the `@root` subvolume at `/mnt` and then the EFI system partition
at `/mnt/boot`.

## Installation and Configuration

## Post-installation Setup
