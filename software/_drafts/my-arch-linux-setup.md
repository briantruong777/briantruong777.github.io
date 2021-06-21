---
---

I've been a big fan of [Arch Linux] for a long time now. Though it is a pain to
set up, it offers a great opportunity to learn exactly how everything fits
together in a Linux system. There's also that pleasant satisfaction when
everything is finally set up exactly the way I like it. In the interest of
making this process easier for both myself and others, I've written down my
personal guide to setting up Arch Linux. Though this guide does set up Arch as
a [VirtualBox] guest, a lot of this applies to my other setups as well.

[Arch Linux]: https://archlinux.org/
[VirtualBox]: https://www.virtualbox.org/

* TOC
{:toc}

## Overview

This guide is intended to be a companion to the official [installation guide]
and not a replacement since I don't plan on covering every detail here nor will
I include the specific commands you need to run. In general, the [ArchWiki] is
an excellent resource and honestly one of the biggest strengths of Arch Linux
so feel free to consult it at any point. It's even useful when you aren't
running Arch.

[installation guide]: https://wiki.archlinux.org/title/Installation_guide
[ArchWiki]: https://wiki.archlinux.org/

Rather than just going through all the steps, I will focus on explaining the
rationale behind my decisions. After all, the benefit of Arch is that you get
to make the choice, and I would hate to completely take that away from you.
Note that this guide is aimed at a virtual machine setup so I've purposefully
gone with a simpler setup since a VM usually doesn't need as much
functionality. Here are some of the key choices I made:

| Boot loader          | [systemd-boot]                            |
| Full disk encryption | [dm-crypt]                                |
| File system          | [btrfs]                                   |
| Window manager       | [i3]                                      |
| Swap                 | [Swap file]                               |
| Networking           | [systemd-networkd] and [systemd-resolved] |
| Firewall             | TODO                                      |

[systemd-boot]: https://wiki.archlinux.org/title/Systemd-boot
[dm-crypt]: https://wiki.archlinux.org/title/Dm-crypt
[btrfs]: https://wiki.archlinux.org/title/btrfs
[i3]: https://wiki.archlinux.org/title/i3
[Swap file]: https://wiki.archlinux.org/title/Swap#Swap_file
[systemd-networkd]: https://wiki.archlinux.org/title/Systemd-networkd
[systemd-resolved]: https://wiki.archlinux.org/title/Systemd-resolved

## Pre-installation

To start off, download an installation image and then create a new VM in
VirtualBox. There are a variety of settings you can configure as you like, but
there are a few recommendations I would make.

First, enable EFI mode. Though it's perfectly fine to stick to BIOS mode, most
modern computers are using UEFI, so you might as well get used to it. If you do
decide to stick with BIOS mode, you'll need to choose a different boot loader
than [systemd-boot] since it only supports UEFI.

Second, for VMs I recommend choosing the disk size wisely. Though it's possible
to resize things later (or add more drives if you use [btrfs]), it's more
convenient to ensure you have enough space from the beginning. Arch can be run
very lightweight, but in my experience, 8 GB is as low as you'd want to go and
still have an acceptable desktop environment. Even then, space will be tight if
you start installing a lot of larger packages (LaTeX, fonts, etc.) and that's
not even including your own files. I personally go with 32 GB so I don't need
to worry too much about it.

Finally, here are some less important options that I like to set.

* Enable the shared clipboard. This won't actually work until we install the
  [VirtualBox Guest Additions], but feel free to enable it early so we don't
  forget.
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
  amount and will likely not be enough for graphics heavy workloads.
* Enable 3D acceleration. This will allow usage of your host system's graphics
  card and will improve performance of programs that can take advantage of that
  (e.g. web browsers).
* If you are storing the VM's hard drive on an SSD, mark the SSD attribute for
  the disk. This will tell the guest system that it is on an SSD so it can
  change it's behavior to take advantage of that.
* Disable showing the mini toolbar in full-screen mode. This is just my
  personal taste, but I prefer not having any VirtualBox UI when in full-screen
  mode.

[VirtualBox Guest Additions]: https://wiki.archlinux.org/title/VirtualBox/Install_Arch_Linux_as_a_guest#Install_the_Guest_Additions
[Physical Address Extension]: https://en.wikipedia.org/wiki/Physical_Address_Extension

Set up the VM to boot from the installation image and then boot it. This will
bring you into a live installation of Arch Linux that you will use to set up
your actual installation. Note that you may want to do everything inside
[`tmux`] so you can scrollback to previous output[^console] and copy/paste
between multiple terminals.

Start off with the first basic steps of:
1. Set the keyboard layout if needed.
1. Set the console font if desired (I like `Lat2-Terminus16` which is just
   [`terminus-font`]).
1. Confirm you are running in UEFI mode.
1. Confirm you have internet access.
1. Enable NTP for the system clock.

[^console]: You may be thinking that the Linux console previously had scrollback capabilities by itself. Don't worry, you're not crazy. [It was removed somewhat recently due to bit rot of the code](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.9-Drops-Soft-Scrollback). Actually, it's possible by the time you are reading this, it was restored back already in which case maybe `tmux` will be less useful.
[`terminus-font`]: https://archlinux.org/packages/community/any/terminus-font/

### Disk Partitioning

There are many ways to partition the disk for a Linux install depending on your
needs. This guide will aim to keep things simple, but before we start, there
are a few things to keep in mind.

First, using [GPT] is recommended over [MBR] for booting in UEFI mode since it
is more flexible and better supported by motherboards and operating systems
implementing UEFI[^mbr]. Second, UEFI requires an [EFI system partition] since
that is where UEFI will look for boot loaders to run. Given that we are trying
to keep it simple, we will only have 2 partitions:

1. EFI system partition
1. Encrypted root directory partition

[GPT]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[MBR]: https://en.wikipedia.org/wiki/Master_boot_record
[^mbr]: The ArchWiki's entry on [partitioning](https://wiki.archlinux.org/title/Partitioning#Choosing_between_GPT_and_MBR) describes these reasons in more detail, but in general, there's little reason to use MBR unless you are stuck with legacy hardware.
[EFI system partition]: https://en.wikipedia.org/wiki/EFI_system_partition

We will not need a swap partition because we will be using a [swap file]
instead. Swap files are convenient to resize (or even delete entirely to free
up space in a pinch). However, if you choose to run [btrfs] and wish to add
more disks in the future, you may want to use a swap partition since
[btrfs doesn't support swap files on filesystems with multiple disks](https://btrfs.wiki.kernel.org/index.php/FAQ#Does_Btrfs_support_swap_files.3F).
On a VM, increasing the disk's size isn't that hard, but on a real machine,
adding another disk is a much simpler option than trying to copy everything on
the smaller disk onto a larger disk.

Though the OS can be in a separate partition, we will keep the kernel image in
the EFI system partition since we will not be encrypting our kernel image. In
theory, encrypting the kernel will prevent an attacker from modifying the
kernel, but in reality this won't stop them since they can just modify the boot
loader instead[^maid]. With this in mind, the EFI system partition is a
convenient alternative, but we could've created another partition just for the
kernel. Note that if you are installing Arch on a system that already has an
EFI system partition, [the existing partition may be too small](https://wiki.archlinux.org/title/Dual_boot_with_Windows#The_EFI_system_partition_created_by_Windows_Setup_is_too_small)
and you may be forced to put the kernel on a separate partition. Either you'll
need to use [systemd-boot's XBOOTLDR](https://wiki.archlinux.org/title/Systemd-boot#Installation_using_XBOOTLDR)
or use a different boot loader.

[^maid]: This attack vector is known as the [Evil Maid Attack]. The correct way to prevent this is to use [Secure Boot], but in my opinion, that is overkill unless you worried about an intelligence agency coming for you.
[Evil Maid Attack]: https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html
[Secure Boot]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

There are many tools that can be used to partition the drive so choose whatever
suits you (I personally just use `fdisk`). The ArchWiki recommends making the
EFI system partition 260 MB which I believe comes from the
[minimum FAT32 partition size on certain drives](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#system-partition).
Use the remainder of the disk as the second partition which will be storing the
root directory.

### Set Up Encryption

We will be using [dm-crypt] to encrypt the entire root directory partition. The
main reason to do this is to prevent someone from accessing your data if your
hard drive is stolen. However, you will need to type in the encryption password
every time you boot the system. For convenience, I usually make the encryption
password the same as my login password which is secure enough for me (note that
by default nothing will keep the passwords in sync).

Following [these instructions](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition),
you should first encrypt the partition via the `cryptsetup` command and set it
up to be mounted at `/dev/mapper/cryptroot`. At this point,
`/dev/mapper/cryptroot` represents the unencrypted drive so anything you write
there will be encrypted. Now, there are still more steps necessary to set up
automatic decryption of the partition at boot time, but we will get to that
later when we set up [initramfs](#initramfs).

### Format Partitions

Next, you will need to format each partition with the correct file system.
[The EFI system partition should be FAT32](https://wiki.archlinux.org/title/EFI_system_partition#Format_the_partition),
but your root directory partition can be whatever you like. If you encrypted
your root directory partition, ensure that you are formatting the unencrypted
mount point (e.g. `/dev/mapper/cryptroot`) rather than the partition itself
otherwise you will destroy the encryption. In this guide, we will be using
[btrfs] since it supports snapshots and automatic compression.

Now before we get any further, I know that btrfs has a pretty bad reputation
for losing data which is pretty much the worst possible thing a file system
could do. This is especially ironic since it is a [copy-on-write] file system
which is traditionally considered to be safer from corruptions since they
easily support atomic operations due to simply never overwriting previous
data/metadata. I believe most of these fears were due to people using btrfs
with a RAID5/6 setup [which even now isn't supported by btrfs](https://btrfs.wiki.kernel.org/index.php/RAID56).
Anyway, I don't have much experience with btrfs, but it has a lot of nice
features and enough people saying good things about it (it's now the
[default for Fedora](https://fedoraproject.org/wiki/Changes/BtrfsByDefault))
for me to justify using it. Still, if its reputation scares you off, feel free
to use a different file system.

[copy-on-write]: https://en.wikipedia.org/wiki/Copy-on-write

### Mounting Filesystems

Next, we need to decide how our newly formatted partitions will be mounted. If
you didn't choose to use [btrfs], the straightforward scheme is to just mount
your new root directory at `/mnt` and then mount the EFI system partition at
`/mnt/boot` (if you don't want to put your kernel image in the EFI system
partition, then do `/mnt/efi` instead). However, with btrfs, we have some
additional options to consider.

First, you should decide whether to mount the btrfs directory with
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
implemented via copying the metadata which makes updating the access time even
more expensive.

[`noatime`]: https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs(5)#NOTES_ON_GENERIC_MOUNT_OPTIONS

#### Subvolume Layout

Before we start mounting things, we should figure out how we want to lay out
our [subvolumes]. Subvolumes act as directory-like mount points that maintain
their own file tree even though they are all part of the same file system. Most
usefully for us, subvolumes support snapshotting. In btrfs, a [snapshot] is
just a subvolume that starts off with the same file tree of another subvolume.
Once the snapshot is created, its file tree is modified independently of the
original subvolume e.g. writes to one subvolume will not affect files in the
other. Cheap snapshotting is one of the main advantages of a [copy-on-write]
file system.

[subvolumes]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes
[snapshot]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots

To start off, there are a [few basic approaches](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout)
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
my documents vs I need to downgrade a package). The `swap` subvolume is for
storing a [swap file] since it requires that the containing subvolume is not
snapshotted or compressed.

To actually create this layout, you should first mount the btrfs top-level
somewhere (e.g. `/btrfs`) so you can `cd` in and start creating the subvolume
and directory structure. To continue with the rest of the installation, you
should mount everything:

* `@root` subvolume at `/mnt`
* `@home` subvolume at `/mnt/home`
* EFI system partition at `/mnt/boot`
* btrfs top-level subvolume at `/mnt/.btrfs`
* `@swap` subvolume at `/mnt/swap` (don't forget to disable compression)

## Installation

Next, you will use `pacstrap` to install the necessary packages to the root
directory of your new system which is mounted at `/mnt`. You should include
any additional packages that would be useful to have while you are setting up
everything else. Some packages I like to install early since they are pretty
helpful even without configuration:

* [`btrfs-progs`]
  * Provides user-space tools for working with btrfs. Note that this package is
    basically a necessity because when you generate the initramfs via
    `mkinitcpio`, it will be unable to run the fsck build hook without this.
* [`man-db` and `man-pages`](https://wiki.archlinux.org/title/Man_page)
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
    shell that includes advance features (syntax highlighting,
    auto-complete[^fish]) without requiring any configuration. However, it is
    not POSIX compliant so scripts that don't have a [shebang] are likely to
    break. If you want something closer to `bash`, [`zsh`] is a good choice,
    but it requires a bit of configuration before it gets to the same level as
    `fish`.

[`btrfs-progs`]: https://wiki.archlinux.org/title/btrfs#Preparation
[`neovim`]: https://wiki.archlinux.org/title/Neovim
[`tmux`]: https://wiki.archlinux.org/title/Tmux
[`fish`]: https://wiki.archlinux.org/title/fish
[shebang]: https://en.wikipedia.org/wiki/Shebang_(Unix)
[`zsh`]: https://wiki.archlinux.org/title/zsh
[^fish]: Note that `fish` usually would parse man pages to generate auto completions for commands it doesn't support out of the box, but that requires installing `python` so consider installing that as well.

Next, you should run `genfstab` as indicated in the installation guide.
Afterwards, you will want to modify the `/etc/fstab` file to remove the
`subvolid=` mount option. This way, you can restore from a snapshot by a simple
`mv` of the `@root` subvolume to `@root_old` and then taking a snapshot of the
desired snapshot and putting it at `/@root`. If you leave the `subvolid=` in,
the mounting will fail since the subvolume ID will be different.

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

Wireless configuration is more complicated. [NetworkManager] is probably the
simplest way to get things working since it has a GUI which makes adding new
wireless networks easy. Otherwise, I don't have much advice here so feel free
to explore other solutions.

[NetworkManager]: https://wiki.archlinux.org/title/NetworkManager

### initramfs

Though `pacstrap` already created an initramfs, it won't actually work for us
since we need to decrypt our root directory partition when we boot. The main
thing we need to do is update the [list of hooks](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio).
Unless you have multiple encrypted drives (or encrypted with a detached LUKS
header), it should be fine to stick with the `encrypt` hook instead of
`sd-encrypt`, but you will need to choose the latter if you do need those
features. `sd-encrypt` will require you to switch out other hooks for their
systemd equivalents.

Don't forget to run `mkinitcpio -P` otherwise your changes won't take effect
and your system will fail to boot!

### Boot Loader

The last thing we need to do before we can boot into our system is set up a
boot loader. There are multiple boot loaders you can choose from, but we will
use [systemd-boot] since it is already included and quite simple. Note that it
only supports UEFI and not BIOS so if you are operating in BIOS mode, you will
need to choose another boot loader.

Be sure to configure things correctly otherwise your system won't boot. For
systemd-boot in particular, you will need to create a loader entry that will
load the OS (see `/usr/share/systemd/bootctl/arch.conf` for an example). You
will also need to ensure you set the correct kernel parameters
for [decrypting the root directory](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader)
and [mounting the correct btrfs subvolume](https://wiki.archlinux.org/title/btrfs#Mounting_subvolume_as_root)
(use the path `/@root` so it's easy to restore a snapshot). I'd also recommend
setting the `editor` option to `no` so that people who boot your system cannot
modify the kernel parameters before logging in (though feel free to do this
later if you think you'll need to change the parameters frequently for initial
setup).

Finally, if you are installing this on a real machine, you should install the
correct [microcode] for your CPU.

[microcode]: https://wiki.archlinux.org/title/Microcode

### Reboot

Assuming nothing went wrong, you should have a functioning Arch Linux system,
so exit the chroot. Feel free to try `umount -R /mnt` to see if there is still
some process busy reading/writing to something (use `fuser` to figure out who).
Finally, you should reboot the machine to see if your new OS boots up (don't
forget to remove the installation image).

If things didn't go so well, you'll need to investigate what happened and try
to fix it. Double check this guide and the [installation guide] to make sure
you followed all the steps. I also recommend reading through the ArchWiki on
topics related to the problem.

## Post-installation

At this point, feel free to close this guide and do your own thing. Your system
can independently boot itself now, and you can install whatever you like via
`pacman`. However, I would not consider this system to be complete since it
needs some work until it's usable and secure enough for daily usage. As you may
have guessed, this is based on the official [general recommendations] page
which is usually what you would follow after finishing the [installation
guide].

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
# btrfs subvolume snapshot -r @root snapshot/root/$(date -Iseconds)
```

Restore from a snapshot by replacing the old subvolume with a new read-write
snapshot of your chosen snapshot. Since we are mounting our subvolumes by path,
you need to ensure the path of the restored snapshot is exactly the same. If
you are replacing the root directory, you should reboot to get the new snapshot
mounted. For the home directory, you could maybe get away with remounting it
live, but it's probably safer to just reboot.

After you've confirmed that everything is good, you can delete the old
subvolume (feel free to keep a snapshot of it). **Do not delete the old root
directory subvolume while it is still mounted.** If you do so, you will lose
your entire root directory and be unable to run any programs including the
`btrfs` command that you would need to fix this.

```
# cd /.btrfs
# mv @root @root_old
# btrfs subvolume snapshot snapshot/root/chosen_snapshot @root
# reboot
# btrfs subvolume delete @root_old
```

Since I always forget all the commands, I usually write a README.md in
`/.btrfs` describing everything. You could even make a script if you like.

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
for a long-running script to run `sudo` and fail if you don't happen to be
around to type in the password. This way, you won't have to restart the entire
script. Other ways of dealing with this (extending `sudo` timeout or not
requiring a password) are insecure.

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
(i.e. no `ssh`) and there are no other users logged in. For other
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

Note that you can still "login" as the root user by doing `sudo -i` to get a
root shell.

### pacman

As you've likely figured out by now, `pacman` works just fine out of the box.
However, there are a few things you'd probably like to set up before you start
installing more things. First off, here are a few options in the `pacman.conf`
file you may be interested in:

* `Color`: Enables color output in the terminal.
* `ParallelDownloads`: Download multiple packages at the same time. The default
  of 5 is probably fine unless it doesn't fully saturate download speed.

`pacman` also supports [pre/post-transaction hooks](https://wiki.archlinux.org/title/Pacman#Hooks)
which allow you to run arbitrary commands before/after `pacman` does anything.
This allows you to take a btrfs snapshot right before you install anything
which makes it easy to revert a bad upgrade. Check out `man alpm-hooks` for
details on the syntax of the hook files. If you don't feel like doing all this
manually, consider using [Snapper] and follow [these instructions](https://wiki.archlinux.org/title/Snapper#Wrapping_pacman_transactions_in_snapshots).

[Snapper]: https://wiki.archlinux.org/title/Snapper

#### Reflector

[Reflector] is a tool that can update the `pacman` mirror list with the best
mirrors based on how up-to-date they are and their speed. Though the default
`/etc/pacman.d/mirrorlist` file may work okay, it will likely not be optimal
for your current location.

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
* `--latest 10`: Find the 10 most up-to-date mirrors. Any mirror that is
  up-to-date enough is acceptable, but might as well pick the latest 10. I
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
risk of locking up your entire system. Also, if you wish to set up
[hibernation], you'll need swap space at least as large as your RAM otherwise
hibernation might fail sometimes if there is too much RAM to save.

[swap]: https://wiki.archlinux.org/title/Swap
[hibernation]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation

I'd recommend setting up a [swap file] since it is way more convenient than a
swap partition (plus it'll be annoying to make space for a new partition now
that we've installed everything). Note that if you are using btrfs, you'll need
to follow [these instructions](https://wiki.archlinux.org/title/Btrfs#Swap_file)
and use the separate `@swap` subvolume to ensure it'll work correctly. Here's
the commands to set up the swap file in btrfs.

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

With the empty swap file created, you need to decide how big you want it.
Usually you'd match the amount of RAM just in case you do decide to set up
hibernation, but for a VM with limited disk space, that might be too much
space. I'd stick with just a few GiB (maybe even just 1 GiB). Feel free to make
it larger since it's pretty easy to change the size later.

### i3

TODO

### Firewall

TODO

### ssh

TODO

### AUR

TODO

## Regular Maintenance

Congratulations! Your Arch Linux system is fully set up and ready for use. If
you are new to Arch, you should read through the [system maintenance] guide. To
sum it up, you need to keep your system up-to-date on a regular basis with
these steps:

1. Check <https://archlinux.org/> for any required manual intervention
1. Run `pacman -Syu`
1. Deal with any `pacnew` or `pacsave` files
1. Reboot

[system maintenance]: https://wiki.archlinux.org/title/System_maintenance

Don't put this off since having to deal with multiple changes requiring manual
intervention can be a pain and puts your system at risk of being permanently
broken. Also, only do an upgrade when you can accept your system being broken
for some time (e.g. don't upgrade right before an important presentation).

## Conclusion

Hopefully you got what you needed out of this guide. Though Arch can be a pain
to set up, many Arch users enjoy having a system that they "built"
themselves[^built]. Whether you decide to join them or not, hopefully you found
this experience valuable.

[^built]: Actually, if you want the real "build-it-yourself" experience, you should consider trying out [Gentoo] or [LFS] (Linux From Scratch). I personally have not tried either of them, but my initial impressions are that they aren't worth the hassle. They do offer more customizability than Arch since they require you to compile everything from source, but that doesn't seem that useful unless you are trying to micro-optimize for your hardware, using an unusual CPU architecture, or want to experience truly building a Linux system yourself.
[Gentoo]: https://www.gentoo.org/
[LFS]: https://www.linuxfromscratch.org/
