---
---

**This post now has a [new version] that supersedes this one.**

[new version]: {% post_url 2025-01-01-my-arch-linux-setup-2.0 %}

---

I've been a big fan of [Arch Linux] for a long time now. Though it is a lot of
work to set up, it offers a great opportunity to learn exactly how everything
fits together in a Linux system. There's also that pleasant satisfaction when
everything is finally set up exactly the way I like it. In the interest of
making this process easier for both myself and others, I've written down my
personal guide to setting up Arch Linux.

[Arch Linux]: https://archlinux.org/

* TOC
{:toc}

## Overview

This guide is intended to be a companion to the official [installation guide]
and not a replacement since I don't plan on covering every detail here nor will
I include the specific commands you need to run. You should follow both guides
side-by-side and follow any linked documentation without prompting from me. In
general, the [ArchWiki] is an excellent resource and honestly one of the
biggest strengths of Arch Linux so feel free to consult it at any point. It's
even useful when you aren't running Arch. In fact, I seriously recommend
abandoning this guide and just following the installation guide by itself, but
chances are that isn't acceptable for you otherwise you wouldn't be here.

[installation guide]: https://wiki.archlinux.org/title/Installation_guide
[ArchWiki]: https://wiki.archlinux.org/

Rather than just going through all the steps, I will focus on explaining the
rationale behind my decisions. After all, the benefit of Arch is that you get
to make the choice, and I would hate to completely take that away from you.
Note that this guide is somewhat aimed at a [VirtualBox] setup so some of the
choices may be non-ideal for a physical machine, though I will still cover some
things that aren't useful for a VM (full-disk encryption). Here are some of the
key choices I made:

| Boot process         | UEFI                                      |
| Partitioning         | `/` (encrypted with [dm-crypt]), `/efi`   |
| Boot loader          | [GRUB]                                    |
| File system          | [btrfs]                                   |
| Window manager       | [i3]                                      |
| Swap                 | [Swap file] or [swap partition]           |
| Networking           | [systemd-networkd] and [systemd-resolved] |
| Firewall             | [ufw]                                     |

[VirtualBox]: https://www.virtualbox.org/
[dm-crypt]: https://wiki.archlinux.org/title/Dm-crypt
[GRUB]: https://wiki.archlinux.org/title/GRUB
[btrfs]: https://wiki.archlinux.org/title/btrfs
[i3]: https://wiki.archlinux.org/title/i3
[Swap file]: https://wiki.archlinux.org/title/Swap#Swap_file
[systemd-networkd]: https://wiki.archlinux.org/title/Systemd-networkd
[systemd-resolved]: https://wiki.archlinux.org/title/Systemd-resolved
[ufw]: https://wiki.archlinux.org/title/Uncomplicated_Firewall

## Pre-installation

To start off, [download an installation image](https://archlinux.org/download/).

### VirtualBox Guest

If you are not installing Arch as a VirtualBox guest, you can skip this
section. First, create a new VM in VirtualBox. There are a variety of settings
you can configure as you like, but there are a few recommendations I would
make.

First, enable EFI mode. Though it's perfectly fine to stick to BIOS mode, most
modern computers are using UEFI, so you might as well get used to it. If you do
decide to stick with BIOS mode, be sure you choose a boot loader that supports
it since some newer boot loaders only support UEFI.

Second, you'll need to choose the disk size. Though it's possible to resize
things later (or add more drives if you use [btrfs]), it's more convenient to
ensure you have enough space from the beginning. Arch can be run very
lightweight, but in my experience, 8 GB is as low as you'd want to go and still
have an acceptable desktop environment. Even then, space will be tight if you
start installing a lot of larger packages (LaTeX, fonts, etc.) and that's not
even including your own files. I personally go with 32 GB so I don't need to
worry too much about it. Obviously you'll need more space if you have lots of
personal files (documents, datasets, videos, etc.).

Finally, here are some less important options that I like to set.

* Enable the shared clipboard. This won't work until we install the [VirtualBox
  Guest Additions], but feel free to enable it early so you don't forget.
* Set the amount of memory. Make sure not to exceed what your host system has
  available.
* Set the boot order. Chances are that the default is perfectly fine, but I
  usually remove everything except the optical drive and the hard disk, and
  then rearrange the optical drive to come first.
* Set the number of processors. You can set this to match the number of cores
  your computer has since the host and guest machine can share all the cores
  safely.
* Enable PAE/NX. This will enable [Physical Address Extension] which includes
  support for the no-execute bit which can provide additional security by
  preventing execution of code in data-only memory pages. I'm not actually sure
  enabling this alone is enough, but might as well do so.
* Set video memory to the maximum (128 MB). This is already a relatively small
  amount and will likely not be enough for graphics heavy workloads, so even a
  basic desktop environment might have trouble with anything less.
* Enable 3D acceleration. This will allow usage of your host system's graphics
  card and will improve performance of programs that can take advantage of that
  (e.g. web browsers).
* If you are storing the VM's hard drive on an SSD, mark the SSD attribute for
  the disk. This will tell the guest system that it is on an SSD so it can
  change its behavior to take advantage of that.
* Disable showing the mini toolbar in full-screen mode. This is just my
  personal taste, but I prefer not having any VirtualBox UI when in full-screen
  mode.

Finally, add the installation image to the VM.

[VirtualBox Guest Additions]: https://wiki.archlinux.org/title/VirtualBox/Install_Arch_Linux_as_a_guest#Install_the_Guest_Additions
[Physical Address Extension]: https://en.wikipedia.org/wiki/Physical_Address_Extension

### Initial Steps

Boot the Arch Linux live installation which you will use to set up your actual
installation. Note that you may want to do everything inside [`tmux`] so you
can scrollback to previous output[^console] and copy/paste between multiple
terminals.

Start off with the first several steps:
1. Set the keyboard layout if needed.
1. Set the console font if desired (I like `Lat2-Terminus16` which is just
   [`terminus-font`]).
1. Confirm you are running in UEFI mode (unless you want to use BIOS instead).
1. Confirm you have internet access.
1. Enable NTP for the system clock.

[^console]: You may be thinking that the Linux console previously had scrollback capabilities by itself. Don't worry, you're not crazy. [It was removed somewhat recently](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.9-Drops-Soft-Scrollback). Actually, it's possible by the time you are reading this, it was restored back already in which case maybe `tmux` will be less useful.
[`terminus-font`]: https://archlinux.org/packages/community/any/terminus-font/

### Disk Partitioning

There are many ways to partition the disk for a Linux install depending on your
needs. This guide will aim to keep things simple, but before we start, there
are a few things to keep in mind.

First, using [GPT] is recommended over [MBR] for booting in UEFI mode since it
is more flexible and better supported by motherboards and operating systems
implementing UEFI[^mbr]. Second, UEFI requires an [EFI system partition] since
that is where UEFI will look for boot loaders to run. Given that we are trying
to keep it simple, we will have 2 main partitions:

1. EFI system partition
1. Encrypted root directory partition

[GPT]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[MBR]: https://en.wikipedia.org/wiki/Master_boot_record
[^mbr]: The ArchWiki's entry on [partitioning](https://wiki.archlinux.org/title/Partitioning#Choosing_between_GPT_and_MBR) describes these reasons in more detail, but in general, there's little reason to use MBR unless you are stuck with legacy hardware.
[EFI system partition]: https://en.wikipedia.org/wiki/EFI_system_partition

Next, you'll need to decide if you want a [swap partition] or a [swap file] (or
none in which case you can skip this). Though swap partitions are the easiest
way to get swap space, they also aren't very flexible since it's harder to
move, shrink, or expand a partition (unless you use [LVM]), and you'll need to
set up encryption for the swap partition separately. In contrast, swap files
can be dealt with just like any other file and will naturally be encrypted if
they are stored on an encrypted partition. However, their setup is more
complicated, and if you choose to run [btrfs] and wish to add more disks in the
future, [btrfs doesn't support swap files on filesystems with multiple disks](https://btrfs.wiki.kernel.org/index.php/FAQ#Does_Btrfs_support_swap_files.3F).

If you want a swap file, then you don't need to do anything else with your
partitioning. Otherwise, you'll need to add a swap partition now. I'd recommend
putting it after your root directory partition so that you can expand/shrink it
by shrinking/expanding your root directory partition and then moving the swap
partition. Otherwise, you'll have to move your root directory partition which
is a huge pain and much riskier.

Note that for a VM, it's probably easier to set up a separate virtual drive
solely for a swap partition since you can resize the drive quite easily.

[swap partition]: https://wiki.archlinux.org/title/swap#Swap_partition
[LVM]: https://wiki.archlinux.org/title/LVM

#### Storing `/boot` on Root Partition

In this guide, we will be keeping our kernel and initramfs (aka the `/boot`
directory) in the root directory partition so that when we take a btrfs
snapshot, it will encompass the entire system including the kernel. This is
important because usually kernel modules are not installed under `/boot`, so if
we kept the kernel in a separate partition, snapshots of the kernel would not
be taken and restored in sync with the kernel modules which will break things.

However, this will cause trouble if you want to encrypt your root partition
since your boot loader will now need to decrypt the partition to get access to
your kernel. Currently [GRUB] is the only boot loader that can unlock a LUKS
partition. If you'd prefer using a different boot loader or generally don't
want to deal with the hassle, then I'd recommend keeping `/boot` unencrypted by
either storing it in the EFI system partition, making a separate boot
partition, or just not encrypting your root partition.

Note that having the kernel encrypted can stop an attacker from directly
modifying the kernel. However, this won't stop a particularly determined
attacker since they can modify the boot loader instead which will give them
access to your system after you decrypt it[^maid].

[^maid]: This attack vector is known as the [Evil Maid Attack]. The correct way to prevent this is to use [Secure Boot] and locking down your BIOS, but in my opinion, that is overkill unless you are worried about an intelligence agency coming for you.
[Evil Maid Attack]: https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html
[Secure Boot]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

#### Running the Partitioning Tool

There are many tools that can be used to partition the drive so choose whatever
suits you (if you use `fdisk`, be sure to switch to GPT). The ArchWiki
recommends making the EFI system partition 260 MB which I believe comes from
the [minimum FAT32 partition size on certain drives](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#system-partition).
Use the remainder of the disk as the second partition which will be storing the
root directory.

### Disk Encryption

Though I don't always do this, this guide will use [dm-crypt] to encrypt the
entire root directory partition. This does cause some pain (e.g. must use
[GRUB] if also encrypting `/boot`, requires password on boot, harder to
configure), but it's really the only way to ensure all your data is safe if
your hard drive is stolen or directly accessed. Otherwise, if you only encrypt
your `/home` partition, it's very easy for personal data to get leaked in
logs/caches or you may accidentally put personal data in an unencrypted
partition.

Now the complicated part. Basically, you want to follow [these instructions](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))
when you encrypt the root partition to ensure [GRUB] can decrypt it. However,
those instructions also include setting up LVM when we just want to set up a
simple partition encryption like in [these instructions](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition).
You'll need to follow some mixture of the two, so be sure to read both
instructions fully before taking any steps. Note that the last steps of the
instructions involve setting up your initramfs and boot loader which we will do
later in this guide so hold onto those steps.

If you decided to go with a separate swap partition, you'll also need to
encrypt that as well. If you want the kernel to resume successfully from it,
you'll also need to configure the swap partition to be decrypted. See [these instructions](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption).

### Format Partitions

Next, you will need to format each partition with the correct file system.
[The EFI system partition should be FAT32](https://wiki.archlinux.org/title/EFI_system_partition#Format_the_partition)
and swap should be formated with `mkswap`, but your root directory partition
can be whatever you like. If you encrypted your root directory partition,
ensure that you are formatting the decrypted mount point (e.g.
`/dev/mapper/cryptroot`) rather than the partition itself otherwise you will
destroy the encryption. In this guide, we will be using [btrfs] since it
supports snapshots and automatic compression.

Now before we get any further, I need to acknowledge that btrfs has a pretty
bad reputation for losing data which is pretty much the worst possible thing a
file system could do. This is especially ironic since it is a [copy-on-write]
file system. Usually such file systems are considered safer from corruptions
since their operations are naturally atomic due to simply never overwriting
previous data/metadata. I believe a lot of the issues were from people using
btrfs with a RAID5/6 setup [which even now isn't supported](https://btrfs.wiki.kernel.org/index.php/RAID56),
but there are definitely enough horror stories to warrant caution. Anyway, I
don't have much experience with btrfs, but it has a lot of nice features and
enough people saying good things about it (it's now the [default for Fedora](https://fedoraproject.org/wiki/Changes/BtrfsByDefault))
for me to justify using it. Still, if its reputation scares you off, feel free
to use a different file system (e.g. [ZFS] is another CoW file system that is
largely considered to be better in every way except it is optimized for
enterprise usage and has unfriendly licensing).

[copy-on-write]: https://en.wikipedia.org/wiki/Copy-on-write
[ZFS]: https://wiki.archlinux.org/title/ZFS

### Mounting Filesystems

Next, we need to decide how our newly formatted partitions will be mounted. If
you didn't choose to use [btrfs], the straightforward scheme is to just mount
your new root directory at `/mnt` and then mount the EFI system partition at
`/mnt/efi` (or at `/mnt/boot` if you want to put your kernel image in the EFI
system partition). However, with btrfs, we have some additional options to
consider.

First, you should decide whether to mount the btrfs directory with
[compression](https://btrfs.wiki.kernel.org/index.php/Compression)
enabled. I don't have any hard data on this, but for a hard drive, enabling
compression is pretty much a no-brainer since disk IO is slow enough to make it
worth the CPU cost to reduce it. For SSDs though, it's probably a performance
hit most of the time. Still, if disk space is more important to you, the
[performance penalty](https://git.kernel.org/pub/scm/linux/kernel/git/mason/linux-btrfs.git/commit/?h=next&id=5c1aab1dd5445ed8bdcdbb575abc1b0d7ee5b2e7)
is likely worth it (e.g. our VM with only 32 GB of space). If performance is
still important, you could choose the LZO algorithm since it is the fastest,
but at that point, you should probably just not enable compression at all (or
even avoid btrfs since it is slower than ext4). If you do decide to enable
compression, you should mount with the option `compress=zstd:1` since the ZSTD
algorithm seems better than the default ZLIB, and level 1 already provides
significant compression so more isn't worth the additional overhead.

Next, you should mount with the [`noatime`] option. In general, this is a
useful optimization for any file system when you don't need access times for
your files, but this is especially true for btrfs since metadata updates are
implemented via copying the metadata which makes updating the access time even
more expensive.

[`noatime`]: https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs(5)#NOTES_ON_GENERIC_MOUNT_OPTIONS

#### Subvolume Layout

Before we start mounting things, we should figure out how we want to lay out
our btrfs [subvolumes] (if you aren't using btrfs, you can skip this section).
Subvolumes act as directory-like mount points that maintain their own file tree
even though they are all part of the same file system. Most usefully for us,
subvolumes support snapshotting. In btrfs, a [snapshot] is just a subvolume
that starts off with the file tree of another subvolume. Once the snapshot is
created, its file tree is modified independently of the original subvolume i.e.
writes to one subvolume will not affect files in the other. Cheap snapshotting
is one of the main advantages of a [copy-on-write] file system.

[subvolumes]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes
[snapshot]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots

To start off, here are a [few basic approaches](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout)
to layout your btrfs subvolumes. Feel free to come up with your own, but here's
the layout of my subvolumes and how they are mounted into the system.
Subvolumes are prefixed with `@` and normal directories are suffixed with `/`.

```
top-level      -> /.btrfs
├── @home      -> /home
├── @root      -> /
├── snapshot/
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
my documents vs I need to downgrade a package). The `@swap` subvolume is for
storing a [swap file] since it requires that the containing subvolume is not
snapshotted or compressed (obviously you can omit this if you don't want a swap
file).

To actually create this layout, you should first mount the btrfs top-level
subvolume somewhere so you can `cd` in and start creating the subvolume and
directory structure. To continue with the rest of the installation, you should
mount everything we've set up. Here's the layout for the above example:

* `@root` subvolume at `/mnt`
* `@home` subvolume at `/mnt/home`
* EFI system partition at `/mnt/efi`
* btrfs top-level subvolume at `/mnt/.btrfs`
* `@swap` subvolume at `/mnt/swap` (we do need to disable compression for that
  subvolume, but doing it via mount options [probably won't work](https://btrfs.wiki.kernel.org/index.php/Compression#Can_I_set_compression_per-subvolume.3F))

## Installation

Next, you will use `pacstrap` to install the necessary packages to the root
directory of your new system which is mounted at `/mnt`. You should include
any additional packages that would be useful to have while you are setting up
everything else. Some packages I like to install early since they are pretty
helpful even without configuration:

* [`btrfs-progs`]
  * Provides user-space tools for working with btrfs. This package is basically
    a necessity because when you generate the initramfs via `mkinitcpio`, it
    will be unable to run the fsck build hook without this.
* [`man-db` and `man-pages`]
  * The ubiquitous `man` command is the standard way to read documentation on
    various commands, programs, system calls, etc. Though most of these can be
    found online nowadays, nothing beats the convenience of having them right
    in your terminal. The `man-db` package provides the `man` command and
    `man-pages` provides Linux specific man pages.
* [`neovim`]
  * Unless you want to use `echo` and `sed`, there is no text editor included,
    so I highly recommend installing one. Obviously you can use whatever text
    editor you like, but `neovim` is a great choice for people who are fans of
    `vim` since it is nearly drop-in compatible but has sensible defaults.
* [`tmux`]
  * Having a terminal multiplexer allows you to have a scrollback buffer in the
    Linux console[^console] which is useful before you get a graphical
    environment set up. It also lets you open multiple terminals and copy/paste
    between them.
* [`fish`]
  * Though chances are you are already familiar with `bash`, `fish` is a nice
    shell that includes advance features (syntax highlighting, auto-complete)
    without requiring any configuration. However, it is not POSIX compliant so
    scripts that don't have a [shebang] are likely to break. Also, you'll
    probably want to include `python` so it can parse man pages for more
    auto-completions. If you want something closer to `bash`, [`zsh`] is a good
    choice, but it requires a bit of configuration before it gets to the same
    level as `fish`.

[`btrfs-progs`]: https://wiki.archlinux.org/title/btrfs#Preparation
[`man-db` and `man-pages`]: https://wiki.archlinux.org/title/Man_page
[`neovim`]: https://wiki.archlinux.org/title/Neovim
[`tmux`]: https://wiki.archlinux.org/title/Tmux
[`fish`]: https://wiki.archlinux.org/title/fish
[shebang]: https://en.wikipedia.org/wiki/Shebang_(Unix)
[`zsh`]: https://wiki.archlinux.org/title/zsh

Next, you should run `genfstab` as indicated in the installation guide. You may
want to first mount your swap partition and anything else you want in the final
system so that they'll end up in the generated `/etc/fstab`. Afterwards, you
will want to modify `/etc/fstab` to remove the `subvolid=` mount option for the
btrfs partition. This way, you can restore from a snapshot by a simple `mv` of
the `@root` subvolume to `@root_old` and then taking a snapshot of the desired
snapshot and naming it `@root`. If you leave the `subvolid=` in, the mounting
will fail since the subvolume ID will be different.

After you've run `arch-chroot`, you will now be acting as if you are inside the
system. You should set up the following next:

1. Time zone
1. Hardware clock
1. Locale
1. Keyboard layout
1. Console font
1. Root password

### Networking

For a VM or a simple wired connection, the networking setup is pretty simple.
The particular details might vary depending on the network card, but just using
the basic [systemd-networkd] should be fine for most cases. You will also need
[systemd-resolved] in order to resolve DNS requests (or an alternative DNS
resolver if you prefer).

Wireless configuration is more complicated. [NetworkManager] is pretty
standard, but I tend to use [iwd] nowadays. Otherwise, I don't have much advice
here so feel free to explore other solutions.

[NetworkManager]: https://wiki.archlinux.org/title/NetworkManager
[iwd]: https://wiki.archlinux.org/title/Iwd

### initramfs

Though `pacstrap` already created an initramfs, it won't actually work if you
need to decrypt the root directory partition when booting since it is missing
some configuration. The main thing we need to do is update the [list of hooks](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio).
Unless you have multiple encrypted partitions (e.g. swap partition) or a
detached LUKS header, it should be fine to stick with the `encrypt` hook
instead of `sd-encrypt`, but you will need to choose the latter if you do need
those features. `sd-encrypt` will require you to switch out other hooks for
their systemd equivalents.

I'd also highly recommend including a key file in the initramfs so it can
unlock the root and swap partition without you having to type in the password
again. Be sure to keep the key file and the initramfs (including the fallback
one) secure by setting the permissions so only the root user can read them.

Don't forget to run `mkinitcpio -P` otherwise your changes won't take effect
and your system will still use the old initramfs likely causing your system to
fail to boot!

### Boot Loader

The last thing we need to do before we can boot into our system is set up a
boot loader. Usually we'd have multiple choices here, but if you need a
bootloader that can both decrypt a LUKS partition and read btrfs, [GRUB] is our
only option. If you are not encrypting your `/boot`, then you can go with a
boot loader that supports btrfs like [rEFInd]. If you aren't using that either,
then [`systemd-boot`] is simple, works pretty well, and is already installed by
default. The rest of this guide will assume you went with GRUB, but I would
recommend going with something simpler if you can.

[rEFInd]: https://wiki.archlinux.org/title/REFInd
[`systemd-boot`]: https://wiki.archlinux.org/title/Systemd-boot

Be sure to configure things correctly otherwise your system won't boot. The
instructions for GRUB are quite complicated. At a high level, you need to do
the following:

1. Set `GRUB_ENABLE_CRYPTODISK=y`
1. Run `grub-install`
1. Configure `/etc/default/grub`
1. Run `grub-mkconfig`

Be sure to double check the resulting `/boot/grub/grub.cfg` file to confirm it
sets the right kernel parameters for [decrypting the root directory](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader)
and [mounting the correct btrfs subvolume](https://wiki.archlinux.org/title/btrfs#Mounting_subvolume_as_root).

Finally, if you are installing this on a real machine, you should install the
correct [microcode] for your CPU.

[microcode]: https://wiki.archlinux.org/title/Microcode

### Reboot

Assuming nothing went wrong, you should have a functioning Arch Linux system,
so exit the chroot. Try running `umount -R /mnt` to see if there is still some
process busy reading/writing to something (use `fuser` to figure out who).
Finally, you should reboot the machine to see if your new OS boots up (don't
forget to remove the installation image and consider changing the boot order to
try the hard disk first).

If things didn't go so well, you'll need to investigate what happened and try
to fix it. Double check this guide and the [installation guide] to make sure
you followed all the steps. I also recommend reading through the ArchWiki on
topics related to the problem.

## Post-installation

At this point, feel free to close this guide and do your own thing. Your system
can independently boot itself now, and you can install whatever you like via
`pacman`. However, I would not consider this system to be complete since it
needs some work until it's usable and secure enough for daily usage. As you may
have guessed, the rest of this guide is based on the official [general
recommendations] page which is usually what you would follow after finishing
the [installation guide].

[general recommendations]: https://wiki.archlinux.org/title/General_recommendations

### Snapshotting with Btrfs

Before we get started, we should go over how to take a snapshot with [btrfs] so
you can restore things if you screw it up. If you aren't using btrfs, this
section doesn't apply to you.

Check out `man btrfs-subvolume` which explains the `snapshot` command. You most
likely want to use the `-r` flag to make the snapshot read-only to avoid
accidentally modifying it later. Taking a snapshot is very easy:

```
# cd /.btrfs
# btrfs subvolume snapshot -r @root snapshot/root/@$(date -Iseconds)
```

Restore from a snapshot by replacing the old subvolume with a new read-write
snapshot of your chosen snapshot. Since we are mounting our subvolumes by path,
you need to ensure the path of the restored snapshot is exactly the same as the
original subvolume. If you are replacing the root directory, you should reboot
to get the new snapshot mounted. For the home directory, you could maybe get
away with remounting it live, but it's probably safer to just reboot.

After you've confirmed that everything is good, you can delete the old
subvolume (feel free to keep a snapshot of it). **Do not delete the old root
directory subvolume while it is still mounted.** If you do so, you will lose
your entire root directory and be unable to run any programs including the
`btrfs` command that you would need to fix this.

```
# cd /.btrfs
# mv @root @root_old
# btrfs subvolume snapshot snapshot/root/@chosen_snapshot @root
# reboot
# btrfs subvolume delete @root_old
```

Since I always forget all the commands, I usually write a README.md in
`/.btrfs` describing all this and include a script for convenience.

### User Account

Currently the only user is the root user which is not great to use regularly
since programs you execute will have much greater permissions than they
probably need. Creating your own user is pretty straightforward using the
`useradd` command, but before you do that, consider setting up the skeleton
directory (defaults to `/etc/skel`) since the files in that directory will be
directly copied to any new user's home directory.

```
# useradd -m -G wheel <username>
# passwd <username>
# chfn <username>
```

`-m` will create a home directory, and `-G` adds the new user to the specified
groups. The [`wheel`] group is an administrative group that gets additional
privileges thanks to [polkit]. `passwd` sets the user's password. `chfn` lets
you set up some additional [GECOS] metadata which is used by some software.

[`wheel`]: https://wiki.archlinux.org/title/Users_and_groups#User_groups
[polkit]: https://wiki.archlinux.org/title/Polkit
[GECOS]: https://en.wikipedia.org/wiki/Gecos_field

#### Sudo

[`sudo`] is a convenient way to run a program with root access without using
`su` to switch to the root user. This is more secure since you can give higher
capabilities only to the commands that need it. Though `sudo` can support more
complicated setups, I find that giving full access to the members of the
`wheel` group is good enough for systems that have one or just a few users.

[`sudo`]: https://wiki.archlinux.org/title/Sudo

Start by installing the `sudo` package. Next, using `visudo` you can modify the
config to allow members of the `wheel` group access to `sudo`.

```
%wheel ALL=(ALL) ALL
```

I also recommend disabling the password input timeout since it is quite common
for a long-running script that runs `sudo` to timeout if you don't happen to be
around to type in the password. With this option enabled, you won't have to
restart the entire script. Other ways of dealing with this (extending `sudo`
timeout or not requiring a password) are insecure.

```
Defaults passwd_timeout=0
```

#### Polkit

With `sudo` set up, you should be able to do any administrative task that you'd
need to do, but for commands that are safe to run (e.g. `poweroff` and
`reboot`), it can be annoying having to type `sudo` and your password every
time. That's where [polkit] comes in. polkit is a framework for normal users to
communicate to privileged programs which essentially allows running certain
privileged commands without `sudo`. Unlike `sudo`, which grants an entire
process root access, polkit only provides access to specific functionality of
privileged programs which makes it more secure.

Start by installing `polkit`. Luckily for us, `poweroff` and `reboot` are
already configured to not require a password if you are physically logged in
(e.g. not `ssh`) and there are no other users logged in. For other
actions/rules, [polkit considers members of the `wheel` group to be administrators](https://wiki.archlinux.org/title/Polkit#Administrator_identities),
so it's best to ensure your administrative accounts are members of `wheel`.

#### Disable Root Login

With both `sudo` and polkit configured, you do not need the root account
anymore, so you can make your system more secure by disabling root login.
Before you do so, log in as your new administrative user and ensure `sudo`
works. Double check [these warnings](https://wiki.archlinux.org/title/Sudo#Disable_root_login)
don't apply to you, and then run the following:

```
# passwd -l root
```

You can still "login" as the root user by doing `sudo -i` to get a root shell.
Note that if something goes wrong during boot, Linux will sometimes drop you in
a root shell, but if you disable root login, this will stop working. If this
worries you, then you should skip this step.

### Pacman

As you've likely figured out by now, `pacman` works just fine out of the box.
However, there are a few things you'd probably like to set up before you start
installing more things. Here are a few options in the `pacman.conf` file you
may be interested in:

* `Color`: Enables color output in the terminal.
* `ParallelDownloads`: Download multiple packages at the same time. The
  commented-out default of 5 is probably fine unless it isn't enough to fully
  saturate your download bandwidth.

`pacman` supports [pre/post-transaction hooks](https://wiki.archlinux.org/title/Pacman#Hooks)
which allow you to run arbitrary commands before/after `pacman` does anything.
You could create a hook to take a btrfs snapshot right before anything happens
(be sure to give it an alphabetically early name to ensure this) which makes it
easy to revert a bad upgrade. Note that the snapshot might be a bit finicky
since it will include `pacman`'s lock file which you'll need to manually
delete. Check out `man alpm-hooks` for details on the syntax of the hook files.
If you don't feel like doing all this manually, consider using [Snapper] and
follow [these instructions](https://wiki.archlinux.org/title/Snapper#Wrapping_pacman_transactions_in_snapshots).

[Snapper]: https://wiki.archlinux.org/title/Snapper

#### Reflector

[Reflector] is a tool that can update the `pacman` mirror list with the best
mirrors based on how fresh they are and their speed. Though the default
`/etc/pacman.d/mirrorlist` file should work pretty well since [it was already generated via Reflector](https://wiki.archlinux.org/title/Installation_guide#Select_the_mirrors),
mirrors do change over time and it's convenient to set it up while you still
have the context.

[Reflector]: https://wiki.archlinux.org/title/Reflector

After you install the `reflector` package, update the
`/etc/xdg/reflector/reflector.conf` to have the parameters you desire. I
personally would recommend setting these options:

* `--protocol https`: The HTTP protocols are generally faster since the
  connection can be reused for multiple downloads. I go with HTTPS for the
  additional security (probably not necessary since packages are signed, but it
  doesn't hurt to be cautious).
* `--country US`: Only consider mirrors in your country since they are likely
  to be close (use `--list-countries` to see the list of countries).
* `--latest 25`: Find the 25 most up-to-date mirrors. Any mirror that is
  up-to-date enough is acceptable, but might as well pick the latest 25. I
  prefer this over `--score` since the score also includes the RTT from the
  Arch Linux servers to the mirror which isn't very useful since your computer
  likely isn't in the same location.
* `--sort rate`: Amongst the selected mirrors, sort them by their speed so
  you'll use the fastest mirror first for downloading packages.

Alternatively you could use `--fastest`, `--age`, and `--sort age` instead of
`--latest` and `--sort rate`. This essentially prioritizes speed over being
up-to-date, but if you pick a low enough `--age`, the result will be similar.

Once the config file is set up, you can start the systemd service
`reflector.service` to update your mirrors. You can take a look at the updated
`/etc/pacmand.d/mirrorlist` file to confirm it worked. Note that you could
enable this service to have it run on every boot, but if you really want to
update your mirrors regularly, I would recommend enabling the systemd timer
(`reflector.timer`) instead since you really don't need to update your mirrors
that often.

### Swap

Though you can probably get pretty far without any [swap] (especially if you
have tons of memory), it's best to have some just in case you exhaust all your
RAM. Otherwise, programs will get killed by the kernel and you'll be at higher
risk of locking up your entire system.

[swap]: https://wiki.archlinux.org/title/Swap

If you wish to set up [hibernation], setting your swap space too small may
cause hibernation to fail if there is too much RAM to save. In practice, the
kernel will do significant compression so there's a decent chance it'll fit
even if your swap space is smaller than your RAM.

[hibernation]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation

#### Swap Partition

If you want to use a [swap partition], hopefully you already created the
partition. Then just follow [these instructions](https://wiki.archlinux.org/title/Swap#Swap_partition).

#### Swap File

If you want to use a [swap file], follow [these instructions](https://wiki.archlinux.org/title/Swap#Swap_file).
Note that if you are using btrfs, you'll need to follow [these instructions](https://wiki.archlinux.org/title/Btrfs#Swap_file)
and use the separate `@swap` subvolume to ensure it'll work correctly. Here's
the commands to set up the swap file in btrfs:

```
# cd /swap
# touch swapfile
# chattr -m swapfile
# chattr +C swapfile
# btrfs property set swapfile compression none
```

Note that the `chattr -m` command shouldn't be necessary since it removes the
flag that disables compression and we want compression disabled, but it seems
you will get an "Invalid argument" error from `chattr +C` if you don't remove
it first (likely a bug).

#### Hibernation

Out of the box, you can use `systemctl suspend` to suspend to RAM (aka
sleep/standby), but if you want to suspend to disk (aka hibernate), you'll need
to set up [hibernation]. If you use a swap partition, it's pretty easy to get
it set up since all you need to do is add the `resume` kernel parameter (see
[these instructions](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Required_kernel_parameters)).
If you have an encrypted swap partition, be sure to point `resume` to the
decrypted mount point or UUID.

If you use a swap file, it'll be a bit more painful since you'll need to
additionally provide the `resume_offset` kernel parameter which will indicate
the offset in the disk that the file is at (see [these instructions](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file)).

### AUR

The [Arch User Repository] is a repository of PKGBUILD scripts that is
maintained by various Arch Linux users. Unlike with packages that come from the
[official repositories], these PKGBUILD scripts are not maintained by trusted
package maintainers meaning anyone can upload a package to the AUR and you
*must not* blindly trust anything from the AUR. Despite this, the AUR is still
very useful for packages that are not popular enough to be in the official
repositories, so there are many users who regularly use packages built from the
AUR. To this end, there are many [AUR helpers] that can make installing
packages from the AUR convenient since usually you'd need to manually download
the PKGBUILD file and any dependencies it needs, build the package, and then
install the package. Doing this once in a while isn't a big deal, but having to
do it every time the package updates is a pain.

[Arch User Repository]: https://wiki.archlinux.org/title/Arch_User_Repository
[official repositories]: https://wiki.archlinux.org/title/Official_repositories
[AUR helpers]: https://wiki.archlinux.org/title/AUR_helpers

Of course, AUR helpers are not in the official repositories since they have the
exact same risks as anything else from the AUR. To install one, you'll have to
manually build it. Follow [these instructions](https://wiki.archlinux.org/title/Arch_User_Repository#Installing_and_upgrading_packages)
carefully to install your AUR helper of choice. There are many to choose from,
but nowadays I usually use [`yay`] or [`paru`]. Note that even if you use an
AUR helper, you should always strive to check the PKGBUILD and other files to
ensure there isn't anything suspicious. Any decent AUR helper will make it
easier by showing you the diff every time the package updates.

[`yay`]: https://github.com/Jguer/yay
[`paru`]: https://github.com/morganamilo/paru

### VirtualBox Guest Additions

If you are installing in VirtualBox, you should install the [VirtualBox Guest
Additions] before you start trying to install any desktop environment or window
manager. This will provide the correct graphics drivers, without which you may
run into issues. Be sure to enable the systemd service so the kernel modules
get started correctly. For additional VirtualBox integration (shared clipboard,
seamless mode), ensure that `VBoxClient-all` is set up to run upon login into
the desktop.

### Window Manager

Assuming you don't want to stay in the Linux Console all the time, you'll want
to install a [desktop environment] or [window manager]. Long story short, if
you just want a standard graphical desktop experience that can be configured
via a GUI and comes with some basic applications, then just install a desktop
environment.

* [GNOME]
* [KDE Plasma]
* [Cinnamon]
* [Xfce]

[desktop environment]: https://wiki.archlinux.org/title/Desktop_environment
[window manager]: https://wiki.archlinux.org/title/Window_manager
[Cinnamon]: https://wiki.archlinux.org/title/Cinnamon
[GNOME]: https://wiki.archlinux.org/title/GNOME
[KDE Plasma]: https://wiki.archlinux.org/title/KDE
[Xfce]: https://wiki.archlinux.org/title/Xfce

Of course, there are more than just these 4, but these are the ones that I've
tried and are relatively popular. My personal choice would be Xfce since it is
lightweight and not too ugly, but GNOME and KDE are definitely more fully
featured.

However, nowadays I use [i3] instead since I find its window tiling scheme
makes a lot of sense and it still supports floating windows. It does take a
while to set up and get used to, but I still like using it. Regardless, you are
obviously free to choose whatever desktop environment or window manager you
like, but if you do go with a window manager, this section may still be useful
since most window managers require setting up similar things.

Before we go any further though, there are a few things you should know. First,
there exists a [Wayland] equivalent of i3 called [Sway]. Wayland is quite
usable now (though Sway doesn't support Nvidia graphics cards), offers better
security, and is aimed at modern-day use cases. However, the [VirtualBox Guest
Additions] doesn't support automatic display resizing with Wayland, so for now,
I stick with i3.

Second, some desktop environments support using an alternative window manager,
so if you like i3 (or any other window manager), but want a desktop experience
that works out of the box, this is definitely an option to consider.

[Wayland]: https://wiki.archlinux.org/title/Wayland
[Sway]: https://wiki.archlinux.org/title/Sway

#### Installation

The [`i3`](https://archlinux.org/groups/x86_64/i3/) package group also includes
some basic packages that you will likely find useful, but you probably don't
want everything in the group since some of the packages provide the same
functionality (and some of them even conflict with each other).

* Window manager: `i3-wm`, [`i3-gaps`]

  `i3-wm` is the normal i3 window manager and technically the only thing you
  need to install to start using i3. `i3-gaps` is a fork that puts aesthetic
  gaps between the windows. Of course, this is literally a waste of space, but
  it does look nice.

[`i3-gaps`]: https://github.com/Airblader/i3

* Status text: [`i3status`], [`i3blocks`], [`i3status-rust`]

  `i3status` generates the status line at the bottom of the screen that
  presents the time, current network status, battery, etc. `i3blocks` and
  `i3status-rust` do the same, but support more functionality like click events
  and running arbitrary commands. `i3status` prides itself on being very fast
  and honestly comes pretty close to having everything I want. However, I can't
  deny the other options look nicer so I usually go with `i3status-rust`.

[`i3status`]: https://i3wm.org/docs/i3status.html
[`i3blocks`]: https://github.com/vivien/i3blocks
[`i3status-rust`]: https://github.com/greshake/i3status-rust

* Screen locker: [`i3lock`], [`i3lock-color`], [`xscreensaver`], [`xsecurelock`]

  Having a screen locker is not necessary for a VM, but for a physical machine,
  you'd definitely want one. The default one is `i3lock` which is pretty basic
  though still neat looking. However, it is missing some ~~basic~~ nicer
  features like displaying the time. `i3lock-color` fixes all that and is more
  customizable, though it does require installing via the AUR. `xscreensaver`
  is a classic and comes with tons of weird graphical spectacles that'll stress
  out your GPU and confuse your coworkers. Finally, there's `xsecurelock` which
  is heavily focused on security (e.g. preventing password bypassing,
  windows/notifications popping up), though if you actually want to keep your
  computer secure, encrypt the entire drive and shut it down or hibernate every
  time you leave it. Anyway, it's honestly overkill for most people, but hey,
  it still supports showing the time unlike `i3lock`, so this is the screen
  locker I go with.

[`i3lock`]: https://i3wm.org/i3lock/
[`i3lock-color`]: https://github.com/Raymo111/i3lock-color
[`xscreensaver`]: https://wiki.archlinux.org/title/XScreenSaver
[`xsecurelock`]: https://github.com/google/xsecurelock

#### Applications

You will likely want to install various applications since a bare bones i3
install has absolutely nothing which may be surprising for people used to other
desktop environments or operating systems. I'd recommend installing at least a
terminal emulator otherwise you will not be able to access a terminal inside i3
(if you forget to do this, you'll need switch to another [Linux Console] to get
access to a terminal).

[Linux Console]: https://wiki.archlinux.org/title/Linux_console

* Display manager: [`xinit`], [`sddm`], [`lightdm`]

  If you installed a full desktop environment, it probably included a [display
  manager] already so you should just use that. Otherwise, you have plenty of
  choices on how to start Xorg. `xinit` is the most manual one since it only
  provides the `startx` command which will start the desktop environment or
  window manager configured in `~/.xinitrc`. As you may have guessed, you'll
  still need to login via the Linux Console to run `startx` which isn't very
  pretty. Also, if you frequently switch desktop environments or window
  managers, modifying a config file each time is annoying, so I'd recommend a
  proper display manager like `sddm` or `lightdm`. Either of them look nice
  enough out of the box, and unlike some other display managers, they are not
  specifically aimed at a certain desktop environment. Choose whatever suits
  your tastes, but I prefer `lightdm` with the `lightdm-gtk-greeter` since it
  looks simple and doesn't have too many dependencies.

[display manager]: https://wiki.archlinux.org/title/Display_manager
[`xinit`]: https://wiki.archlinux.org/title/Xinit
[`sddm`]: https://wiki.archlinux.org/title/SDDM
[`lightdm`]: https://wiki.archlinux.org/title/LightDM

* Terminal emulator: [`gnome-terminal`], [`xfce4-terminal`], [`terminator`],
  [`alacritty`], [`kitty`], [`cool-retro-term`]

  There are tons of terminal emulators you can use, so I'm not going to even
  attempt to describe the different varieties, but here are some that I've
  personally used that work for me. `gnome-terminal` is a [VTE-based] terminal
  that is designed for the GNOME desktop environment, but it works well enough
  here (assuming you don't mind all its dependencies). It does pretty much
  everything you'd expect from a modern terminal. `xfce4-terminal` is another
  VTE-based terminal but this time intended for the Xfce desktop environment.
  `terminator` is yet another VTE-based terminal that isn't intended for any
  particular desktop environment and includes plenty of features.

  Nowadays, GPU accelerated terminals are all the rage (because obviously you
  need to be able to see text streaming at 144 fps), though I should warn you
  that if your hardware doesn't have modern OpenGL support (e.g. VirtualBox),
  these may not work well. `alacritty` is the default for [Sway], written in
  Rust (if that matters to you), and is the officially sanctioned replacement
  for `termite` (my previous chosen terminal). `kitty` is another GPU
  accelerated terminal that offers a lot of flexibility with a variety of
  plugins. Finally, `cool-retro-term` is a fun terminal that lets you time
  travel to the past when screen burn-in was a thing and everything was
  monochrome.

[`gnome-terminal`]: https://wiki.gnome.org/Apps/Terminal
[`xfce4-terminal`]: https://docs.xfce.org/apps/terminal/start
[`terminator`]: https://wiki.archlinux.org/title/Terminator
[`alacritty`]: https://wiki.archlinux.org/title/Alacritty
[`kitty`]: https://wiki.archlinux.org/title/Kitty
[`cool-retro-term`]: https://wiki.archlinux.org/title/Cool-retro-term
[VTE-based]: https://gitlab.gnome.org/GNOME/vte

* Application launcher: [`dmenu`], [`rofi`]

  `dmenu` is the default application launcher and it works perfectly fine even
  though it is a bit lacking in features. Out of the box, it will include
  terminal programs as well which I find annoying since I would've opened up a
  terminal if I wanted one of those. You can use the included
  `i3-dmenu-desktop` script to instead only show programs with a desktop entry.
  An alternative application launcher is `rofi` which has some more features
  like showing icons, supporting themes, and switching applications (its
  original purpose). Other than those, there are plenty of alternative
  application launchers so choose what you like. I personally go with `rofi`
  since it has icons.

[`dmenu`]: https://wiki.archlinux.org/title/dmenu
[`rofi`]: https://wiki.archlinux.org/title/Rofi

* Font: [`noto-fonts`], [`ttf-dejavu`], [`ttf-hack`], [`terminus-font`],
  [`gohufont`]

  Not sure if I should bother having this section since font preference is
  almost purely personal taste, but here we go anyway.

  To ensure you have decent coverage, you'll want to start with a font that
  [provides a base font set](https://wiki.archlinux.org/title/Font_package_guidelines#Provides).
  `ttf-dejavu` is an extension of [Bitstream Vera](https://en.wikipedia.org/wiki/Bitstream_Vera)
  intended to provide more Unicode coverage. It also includes a decent
  monospace font with some thought paid to programming. Alternatively,
  `noto-fonts` is a Google commissioned font that intends to cover all of
  Unicode so there will be ["no more tofu"](https://www.google.com/get/noto/).
  If you install its additional packages, you'll really get almost full
  coverage, but it will take up a lot of space.

  `ttf-hack` is a nice monospace font that is derived from Bitstream Vera and
  DejaVu, but includes some more programming specific adjustments. If you
  prefer a bitmap font, you can try out `terminus-font` (the same font as
  `Lat2-Terminus16` in the Linux Console) or `gohufont` from the AUR. However,
  bitmap fonts are falling out of favor so you may have to struggle to get them
  to work.

  Regardless of which font you pick, you may need to do some [font
  configuration] to get things working properly.

[`noto-fonts`]: https://www.google.com/get/noto/
[`ttf-dejavu`]: https://dejavu-fonts.github.io/
[`ttf-hack`]: https://sourcefoundry.org/hack/
[`terminus-font`]: http://terminus-font.sourceforge.net/
[`gohufont`]: https://font.gohu.org/
[font configuration]: https://wiki.archlinux.org/title/font_configuration

* Notification server: [`dunst`]

  Without a notification server, you won't be able to see any notifications.
  `dunst` is a lightweight notification server that allows some basic
  appearance customization, notification history, and even scripting based on
  the incoming notifications. You can also use notification servers
  specifically intended for a desktop environment, but I find that `dunst` gets
  the job done well enough.

[`dunst`]: https://wiki.archlinux.org/title/Dunst

### Backup Kernel

Since Arch has a [rolling release] model, packages are upgraded promptly after
the upstream source is. Though packages generally go through the [testing
repository] first, the chances of something breaking are still higher than
usual. This can especially be troublesome if a kernel upgrade breaks your
system since generally you won't be able to boot anymore, and you'll have to
resort to booting a live installation in order to fix things.

[rolling release]: https://en.wikipedia.org/wiki/Rolling_release
[testing repository]: https://wiki.archlinux.org/title/Official_repositories#testing

With this in mind, I'd recommend also installing the [longterm release kernel].
This kernel will still receive bug fixes and security updates, but not any new
features present in the latest stable version. For Arch, you can install it via
the [`linux-lts`] package. Note that this package runs the latest longterm
release, so it still gets upgraded whenever a new longterm release is chosen
(approximately once a year).

[longterm release kernel]: https://www.kernel.org/category/releases.html
[`linux-lts`]: https://archlinux.org/packages/core/x86_64/linux-lts/

Once you've installed the `linux-lts` package, you'll need to set up your boot
loader to have additional entries for the `linux-lts` kernel. By default, the
`mkinitcpio` hook should create new `vmlinuz-linux-lts` and
`initramfs-linux-lts` files (as well as fallback versions) that you can refer
to in the boot loader entry. GRUB's `grub-mkconfig` will actually detect the
additional kernel versions for you.

Note that using btrfs mitigates the need to keep a backup kernel (assuming your
kernel is stored on the btrfs file system of course) since you can revert to an
older version if something breaks. However, you may still want to keep a backup
kernel anyway since you might not even be able to boot using the broken kernel.

### Firewall

Linux doesn't exactly need a firewall since ports are closed by default, but if
you'd like to be a bit safer, I'd still recommend having at least a basic one
set up. This is especially true if you run any services that accept incoming
connections (ssh, Samba) or your system is exposed on the public internet.
Though you can manage [`nftables`] manually, I find that using [ufw] works
pretty well and is much simpler. It generally has acceptable defaults, but
you'll need to add exceptions for any services you wish to expose. If you can
only access your system remotely, be sure to set up the exception for ssh
carefully otherwise you may lock yourself out.

[`nftables`]: https://wiki.archlinux.org/title/nftables

### Hardware

If you are not installing in a virtual machine, you'll likely need to look at
these additional guides to set up your specific hardware:

* Graphics card
  * [NVIDIA](https://wiki.archlinux.org/title/NVIDIA)
  * [AMD](https://wiki.archlinux.org/title/AMDGPU)
  * [Intel](https://wiki.archlinux.org/title/Intel_graphics)
* [Audio](https://wiki.archlinux.org/title/sound_system)
* [Laptop](https://wiki.archlinux.org/title/laptop)

## Regular Maintenance

Congratulations! Your Arch Linux system is fully set up and ready for use
(well, according to my standards at least). If you are new to Arch, you should
read through the [system maintenance] guide. To sum it up, the main thing you
need to do is keep your system up-to-date on a regular basis with these steps:

1. Check [archlinux.org](https://archlinux.org/) for any required manual
   intervention
1. Run `pacman -Syu`
1. Deal with any `pacnew` or `pacsave` files
1. Reboot
1. Confirm all your systemd services are working with `systemctl status`

[system maintenance]: https://wiki.archlinux.org/title/System_maintenance

Don't put this off since having to deal with multiple changes requiring manual
intervention can be a pain and puts your system at higher risk of being
permanently broken. Also, only do an upgrade when you can accept your system
being broken for some time (e.g. don't upgrade right before an important
presentation).

## Conclusion

Hopefully you got what you needed out of this guide. Though Arch can be a pain
to set up, many Arch users enjoy having a system that they "built"
themselves[^built]. Whether you decide to join us or not, hopefully you found
this experience valuable.

[^built]: Actually, if you want the real "build-it-yourself" experience, you should consider trying out [Gentoo] or [LFS] (Linux From Scratch). I personally have not tried either of them, but my initial impressions are that they aren't worth the hassle. They do offer more customizability than Arch since they require you to compile everything from source, but that doesn't seem that useful unless you are trying to micro-optimize for your hardware, using an unusual CPU architecture, or want to experience truly building a Linux system yourself.
[Gentoo]: https://www.gentoo.org/
[LFS]: https://www.linuxfromscratch.org/
