---
---

This is the next version of my [original guide] to setting up [Arch Linux]. Why
am I writing this? Well, I'm getting a new [Framework Laptop 13], and I want to
install Arch Linux on it. Yup, that's pretty much it. Some of my preferences
have changed, but don't be surprised if much of this feels the same.

[original guide]: {% link _posts/2022-01-17-my-arch-linux-setup.md %}
[Arch Linux]: https://archlinux.org
[Framework Laptop 13]: https://frame.work

-   TOC
{:toc}

## Overview

I'm pretty sure I'm the only one who will ever read this post, but just in case
that isn't true, here's a brief overview of things.

This guide is a companion to the official [installation guide], i.e., you
should have them open and read them side-by-side. Though I'll try to cover
every single step, this guide will likely fall out-of-date, so double check
everything. I will also not prompt you to further research anything nor provide
any exact commands. In general, the ArchWiki is an excellent resource and you
should get used to using it. This guide mainly serves to call out specific
parts and explain certain choices.

[installation guide]: https://wiki.archlinux.org/title/Installation_guide

Compared to the original post, the differences include my reduced patience for
customization, unencrypted `/boot`, separate `/efi`, no mention of
virtualization, additional laptop guidance, and more. Here's an overview of the
setup:

-   [UEFI] with [Secure Boot] via [shim]
-   [rEFInd] boot manager
-   Partitioning ([GPT])
    -   [EFI system partition] mounted to `/efi`
        -   With [SystemRescue] toolkit installed
    -   [dm-crypt] encrypted [LVM] partition
        -   `swap` logical volume
        -   `btrfs` logical volume formatted with [Btrfs]
            -   `@root` subvolume mounted to `/`
                -   [Snapper] snapshots in `/.snapshots`
            -   `@home` subvolume mounted to `/home`
                -   [Snapper] snapshots in `/home/.snapshots`
-   [Hibernation] (suspend-to-disk)
-   Programs
    -   [NetworkManager]
    -   [Ufw] (Uncomplicated Firewall)
    -   [systemd-timesyncd]
    -   [Restic] and [resticprofile] for backups
    -   [Paru] for installing from the [AUR]
    -   [GNOME] desktop environment
    -   [fish] (friendly interactive shell)

[UEFI]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface
[SystemRescue]: https://www.system-rescue.org
[Snapper]: https://wiki.archlinux.org/title/Snapper
[Hibernation]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation
[NetworkManager]: https://wiki.archlinux.org/title/NetworkManager
[Ufw]: https://wiki.archlinux.org/title/Uncomplicated_Firewall
[systemd-timesyncd]: https://wiki.archlinux.org/title/Systemd-timesyncd
[Restic]: https://wiki.archlinux.org/title/Restic
[resticprofile]: https://creativeprojects.github.io/resticprofile
[Paru]: https://github.com/Morganamilo/paru
[AUR]: https://wiki.archlinux.org/title/Arch_User_Repository
[GNOME]: https://wiki.archlinux.org/title/GNOME
[fish]: https://wiki.archlinux.org/title/Fish

## Pre-installation

The main steps from the [installation guide]:

1.  [Download] the installation image and write it to a [USB drive]
2.  Boot the live environment from the USB drive
    -   You may need to disable [Secure Boot] first
    -   You may want to use [tmux] to allow scrolling back to previous output
3.  Set the keyboard layout
4.  Connect to the internet
5.  Update the system clock

[Download]: https://archlinux.org/download
[USB drive]: https://wiki.archlinux.org/title/USB_flash_installation_medium
[tmux]: https://wiki.archlinux.org/title/Tmux

### Partitioning

Your choice of partitioning setup and file system are the first of the many
major decisions you will have to make. These have huge ramification for the
rest of the system, and even worse, it'll be a huge pain to fix later. I'd
recommend taking some time to think it through; otherwise you might find
yourself having to throw away everything and start over from scratch.

Compared to the original guide, the main difference is the unencrypted `/boot`
and mounting the [EFI system partition] at `/efi`. This prevents synchronously
snapshotting the kernel with the rest of the root file system, but since the
original guide was written, I've never had a need to atomically rollback the
kernel and kernel modules together.[^atomic] Thus, it doesn't seem worth the
complexity and suffering from being restricted to [GRUB]. Plus, I am now going
to use [Secure Boot], so the other benefit of preventing tampering of the
kernel is somewhat covered.

[^atomic]: If this is something you are interested in, I'd suggest using a
    distro actually designed for atomic updates/rollbacks like [Fedora
    Silverblue] or [NixOS].

[Fedora Silverblue]: https://fedoraproject.org/atomic-desktops/silverblue
[NixOS]: https://nixos.org

And yes, you heard that right, I'm using [Secure Boot]. I used to think it was
overkill unless some nation's intelligence agency is gunning for you, but
malware that infects your boot loader or kernel is not as rare as it used to
be. Plus, it is supported by both Ubuntu and Fedora which makes it commonplace
even for Linux machines.

The other major difference from the original guide is that I use [LVM] to
create separate logical volumes for the [Swap] space and [Btrfs] file system. I
used to suggest using a Swap file instead, but it's a bit finicky to set up on
[Btrfs], and [LVM] provides the equivalent convenience of being able to resize
it on-the-fly. Plus, having a single [LVM] partition means we can
encrypt/decrypt the Swap and root file system in one fell swoop. Finally, [LVM]
gives much more flexibility in the future for partitioning shenanigans.

A minor difference from the original guide is that I use [Snapper] to
automatically take [Btrfs] snapshots, so the layout of [Btrfs] subvolumes is a
bit different. Note that if you want convenient rollback, you may want to
follow the [suggested subvolume layout] or even set up [GRUB] to allow choosing
a snapshot to boot into. I don't bother since I've never needed to do a full
rollback like that (Arch Linux is surprisingly reliable for me), and if the
need arises, I don't mind doing it manually (that's what the [SystemRescue]
toolkit is for).

[suggested subvolume layout]: https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout

### Create partitions

We will only need 2 partitions since the primary partition will have [LVM]
setup. Be sure to use the [GPT] partitioning scheme which is the standard
nowadays despite `fdisk` defaulting to MBR (or just use `gdisk` or `cfdisk`).

1.  [EFI system partition] (2 GiB)
2.  Primary partition (remaining space)

[GPT]: https://wiki.archlinux.org/title/Partitioning#GUID_Partition_Table
[EFI system partition]: https://wiki.archlinux.org/title/EFI_system_partition

All [UEFI] systems require an [EFI system partition]. In our case, we are going
to be mounting this partition as `/efi` on our system. We will store our kernel
there (and later on [SystemRescue] installation). This means we need a decent
amount of space in there, so I'd recommend making the partition at least 2 GiB.
You could go even further if you are paranoid. Don't forget to format the EFI
partition as FAT32.

### Encryption

For our primary partition, we first need to encrypt it using [dm-crypt]. There
are [many possible encryption setups], but the one we will use is [LVM on LUKS]
for the reasons described earlier.

[dm-crypt]: https://wiki.archlinux.org/title/Dm-crypt
[many possible encryption setups]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system
[LVM on LUKS]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

Since I just want to encrypt things to ensure a thief can't access my personal
data, I enable [dm-crypt TRIM support] just in case it helps optimize the SSD's
performance or lifetime. If you are paranoid about someone uncovering sensitive
information from the pattern of TRIM'd blocks, then leave things on the default
settings which don't do this. That being said though, if you are that paranoid,
you should consider using something like a [detached LUKS header] so there
truly is nothing discernible from drive.

[dm-crypt TRIM support]: https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)
[detached LUKS header]: https://wiki.archlinux.org/title/Dm-crypt/Specialties#Encrypted_system_using_a_detached_LUKS_header

In addition, we can [disable using a read/write work queue]. The queues were
optimizations for batching reads/writes on HDDs, but don't make sense on SSDs
and seem to actively harm performance.

[disable using a read/write work queue]: https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance

### LVM

Now create an [LVM] physical volume on the `/dev/mapper/` entry for the
encrypted partition, and put it in a volume group. Then create two logical
volumes in the volume group:

[LVM]: https://wiki.archlinux.org/title/LVM

1.  [Swap] logical volume
2.  [Btrfs] logical volume

Set up and enable [Swap] the appropriate logical volume. [Btrfs] will be a bit
more complicated.

[Swap]: https://wiki.archlinux.org/title/Swap

### Btrfs

In case you are not aware of it, [Btrfs] is a CoW (copy-on-write) file system
that has a wide variety of features including utilizing multiple devices,
efficient snapshotting, and transparent compression. These features do come at
a performance cost, and some of them are downright broken like [RAID5/6], but
at least for my purposes, the good outweighs the bad.

[Btrfs]: https://wiki.archlinux.org/title/Btrfs
[RAID5/6]: https://btrfs.readthedocs.io/en/stable/btrfs-man5.html#raid56-status-and-recommended-practices

One big weakness of [Btrfs] is that it behaves poorly when out of disk space
since even deletions require writing new bytes. Such situations are more likely
than one might expect thanks to the convenience of snapshots.

Another weakness is the poor performance for workloads that need to update
existing blocks repeatedly like databases and VMs. Thankfully, [Btrfs] supports
[disabling CoW] for specified directories and files which mitigates the issue.

[disabling CoW]: https://wiki.archlinux.org/title/Btrfs#Disabling_CoW

Anyway, assuming you are going along with using [Btrfs], the main thing you
need to decide is which [RAID profile] to use. If you have a multi-device
system, I assume you intend on using `raid1` or similar for both metadata
(`-m`) and data (`-d`), so be sure to set this when calling `mkfs.btrfs`. For a
single device, `mkfs.btrfs` defaults to `dup` for metadata and `single` for
data. This gives metadata some redundancy to prevent corruptions from breaking
the entire file system.

[RAID profile]: https://btrfs.readthedocs.io/en/stable/mkfs.btrfs.html#profiles

When mounting the [Btrfs] partition, I recommend setting the `noatime` option.
This will skip updating access times of any directories or files accessed.
Though this is beneficial with any file system when you don't care about access
times, it is especially beneficial for [Btrfs] since the entire metadata block
would need to be copied just to update the access time.

The other recommended mount option is `compress=lzo` which makes [Btrfs]
automatically compress all files. Note that there are some basic heuristics to
store a file uncompressed if the compression ratio is poor (e.g. the data was
already compressed), so it's okay to just default to doing compression. There
are other compression algorithms you could choose if you don't mind spending
more CPU (`compress=zstd=15`), but one [random person's benchmark] suggests
that for fast NVMe SSDs, even `zstd=1` is too slow to be beneficial, so `lzo`
it is.

[random person's benchmark]: https://gist.github.com/braindevices/fde49c6a8f6b9aaf563fb977562aafec

Now that all the long-winded explanations are over with, go ahead and create
the following [Btrfs] subvolumes:

-   `@root` mounted at `/`
-   `@home` mounted at `/home`

We keep `/home` in a separate subvolume to ensure that snapshots of system
files don't include user files so rollbacks won't revert user files too. For
the same reason, you could create a subvolume specifically for snapshots too.
You may also want to create separate subvolumes for disabling CoW in certain
directories since [snapshotting can interfere] with that.

[snapshotting can interfere]: https://wiki.archlinux.org/title/Btrfs#Effect_on_snapshots

## Installation

After all that, we can finally start installing things. At this point, double
check you have the following mounted (names will differ based on your setup):

|                        | Path                    | Mounted At             |
| ---------------------- | ----------------------- | ---------------------- |
| [dm-crypt]             | `/dev/nvme0n1p2`        | `/dev/mapper/cryptlvm` |
| [LVM]                  | `/dev/mapper/cryptlvm`  | `/dev/MainVolGrp`      |
| [Swap]                 | `/dev/MainVolGrp/swap`  | Enabled using `swapon` |
| [Btrfs] `@root`        | `/dev/MainVolGrp/btrfs` | `/mnt`                 |
| [Btrfs] `@home`        | `/dev/MainVolGrp/btrfs` | `/mnt/home`            |
| [EFI system partition] | `/dev/nvme0n1p1`        | `/mnt/efi`             |

With everything mounted correctly, go ahead and use `pacstrap` to install the
base system along with any additional packages you'll want to be available. Be
sure to install everything needed to have internet access otherwise you won't
be able to download any additional packages after booting into the system. To
start you off, here's the packages I like to install early, but obviously you
can omit them or choose your own equivalents:

-   Relevant firmware, CPU microcode, etc.
-   `lvm2`
    -   Tools for [LVM]
-   `btrfs-progs`
    -   Tools for [Btrfs]
-   `sbctl`
    -   Tool for [Secure Boot]
-   `sbsigntools`
    -   More tooling for [Secure Boot]
    -   Mainly needed for [rEFInd] to sign its binaries for you
-   `mokutil`
    -   Tool for updating [shim]'s database
-   `refind`
    -   [rEFInd], our boot manager of choice
-   `networkmanager`
    -   All-in-one network management program
-   `base-devel`
    -   Tools needed for compiling packages
-   `man-db` and `man-pages`
    -   man pages
-   `tmux`
    -   Terminal multiplexer
    -   Enables multiple terminals before you have a desktop environment
-   `neovim`
    -   Fork of `vim` that has better defaults
-   `fish` (also `python` for automatic parsing of man pages)
    -   Friendly interactive shell with nice default behavior

## Configure the system

And now the fun part where you have to configure everything correctly otherwise
your system won't boot properly and you will have no clue what the heck went
wrong so you desperately try rerunning all sorts of commands until you are
forced to confront the fact that you have no option left except to delete
everything and start over from the beginning. But anyway, I'm sure that won't
happen to you, so no need to worry.

Use `genfstab` to generate the fstab file at `/mnt/etc/fstab`. `genfstab` uses
the current mountings, so you may need to manually correct the entries. In
particular, the [Btrfs] mountings should use the `subvol=` option to specify
whether to mount `@root` or `@home`. Afterwards, `arch-chroot` into `/mnt` and
configure the following:

1.  Timezone
2.  Hardware clock
3.  Localization
4.  Keyboard layout
5.  Hostname
6.  Network access
7.  Root password

And now we get to the trickiest part, making sure your system can actually boot
on its own. However, we first need to talk about [Secure Boot].

### Secure Boot

[Secure Boot] is a [UEFI] feature that protects against executing unapproved
binaries during the boot process. This mainly serves to prevent malware from
remotely installing itself inside the boot loader, boot manager, or kernel
where it can easily hide from your OS. Note that [Secure Boot] can simply be
disabled in the [UEFI] interface, so it is primarily a deterrent for remote
attackers who cannot access the pre-boot environment.[^uefi-password]

[Secure Boot]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot

[^uefi-password]: Even if you add a BIOS/UEFI password, many machines have
    password reset procedures that only require physical access. Even ignoring
    that, there are a variety of ways to forcibly manipulate or dump the data
    stored in a motherboard. If you are serious about stopping an attacker with
    physical access, the best way is to never allow them physical access in the
    first place.

[Secure Boot] is a complicated beast, but if you are seeking a deeper
understanding, I found Rod Smith's [Dealing with Secure Boot] to be very useful
for both educational and practical purposes. In comparison, this guide will
contain only basic explanations and gloss over many details.

[Dealing with Secure Boot]: https://www.rodsbooks.com/efi-bootloaders/secureboot.html

At its core, [Secure Boot] means that your computer's [UEFI] will only boot
binaries or accept keys that have been approved by Microsoft. This would
usually lock out Linux entirely, but there are specific third-party binaries
that Microsoft has signed which allow for booting other binaries. The two that
matter for us are:

1.  [shim]
2.  [PreLoader]

[shim]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#shim
[PreLoader]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#PreLoader

Both of these allow end-users to add additional binaries and keys beyond the
Microsoft/vendor ones. This does weaken the chain of trust since you might get
tricked into adding an attacker controlled key, or they might replace your
entire boot setup with one that does what they want.[^weak-chain] However,
since this can only be done during the pre-boot process, it'd be hard for a
remote attacker to do so. Of the two, [shim] is the more flexible and
up-to-date one, so I recommended using it over [PreLoader].

[^weak-chain]: If you are worried about this, an alternative is to [replace the
    Microsoft/vendor keys] with your own so that only binaries you sign can be
    executed. This prevents any version of the [shim] or [PreLoader] from
    executing which prevents attackers from supplanting your boot setup with
    their own. However, this could brick your machine if it needs to execute
    firmware signed only with Microsoft/vendor keys. You have been warned.
    Supposedly the Framework Laptop 13 (AMD 7040) supports Secure Boot [without
    any Microsoft/vendor keys], but it may prevent applying [future firmware
    updates].

[replace the Microsoft/vendor keys]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys
[without any Microsoft/vendor keys]: https://community.frame.work/t/solved-secure-boot-and-custom-keys-on-the-amd-motherboard/38362
[future firmware updates]: https://community.frame.work/t/secureboot-setup-mode/14889

The way [shim] works is that:

1.  [UEFI] starts the [shim]
2.  [shim] starts your boot manager
3.  Your boot manager boots your kernel

#1 passes [Secure Boot] restrictions since the [shim] was signed with
Microsoft's key. #2 and #3 require your boot manager and kernel to have their
hashes added to the [shim]'s database or be signed by a Machine Owner Key (MOK)
also added to the [shim]'s database. Note that the former is not recommended
for binaries that are updated frequently (e.g. the Arch Linux kernel) since
you'll eventually run out of NVRAM space and have to remove previous entries
manually.

The MOK is usually generated by you and kept somewhere safe otherwise an
attacker could use it to sign their own binaries. In this setup, we do take a
bit of a risk and leave it in your encrypted root file system. This is way more
convenient for updating the kernel since the MOK will be easily available to
sign the new kernel. However, an attacker could potentially gain access to the
MOK if they gain root access (I assume you would make the key only accessible
to the root user, right?).

Be sure to use a boot manager that actually enforces the same [Secure Boot]
requirements otherwise it'll defeat the point of this whole exercise (e.g.
[rEFInd]).

#### Tooling

`sbctl` is a very nice tool for working with your computer's [Secure Boot]
settings. That being said, be careful you don't accidentally break your system
with it.

Installing `sbctl` also adds pacman and [mkinitcpio] hooks that will sign
updated binaries and the [unified kernel image] automatically. However, you
need to make sure you create/add your keys to `sbctl` properly.

The main commands you'll use:

-   `sbctl status`
    -   Reports the current [Secure Boot] state of your computer
-   `sbctl verify`
    -   Lists relevant binaries and whether they are signed or not
-   `sbctl create-keys`
    -   Creates keys to be enrolled in [UEFI] and used to sign binaries
-   `sbctl import-keys`
    -   Import keys rather than creating them
-   `sbctl sign`
    -   Sign binaries with the created keys
    -   The [sbctl mkinitcpio hook] calls this
-   `sbctl sign-all`
    -   Sign all binaries that were previously passed to `sbctl sign --save`
    -   The [sbctl pacman hook] calls this
-   `sbctl enroll-keys`
    -   Add the created keys to [UEFI]
    -   If you are using the [shim], you don't need this since you will leave
        the existing [UEFI] keys alone
    -   Be careful with this as you could break your system if you use it
        incorrectly

[sbctl mkinitcpio hook]: https://github.com/Foxboron/sbctl/blob/master/contrib/mkinitcpio/sbctl
[sbctl pacman hook]: https://github.com/Foxboron/sbctl/blob/master/contrib/pacman/ZZ-sbctl.hook

There's also the `sbsigntools` package which has various tools including
`sbsign` that is pretty commonly used. In particular [rEFInd] will use `sbsign`
to sign itself. I find `sbctl` is more user-friendly and comes with useful
[mkinitcpio] and pacman hooks, so I really only install `sbsigntools` for the
integration with [rEFInd].

#### Downloading the shim

Like with any program that's been around a while, there's multiple copies and
versions of the [shim] binaries. More importantly, we need one that has been
signed with Microsoft's key otherwise [Secure Boot] won't allow it to run. You
can download one via the [shim-signed] package in the [AUR]. Usually I'd use an
[AUR] helper like [paru] to install the package, but it's a bit of a pain to
set up this early. You can build it manually using something like:

```sh
$ cd /tmp
$ curl -O https://aur.archlinux.org/cgit/aur.git/snapshot/shim-signed.tar.gz
$ tar -xzf shim-signed.tar.gz
$ cd shim-signed
$ runuser -u nobody makepkg -i
```

[shim-signed]: https://aur.archlinux.org/packages/shim-signed/
[paru]: https://github.com/Morganamilo/paru

### Initramfs

[Initramfs] contains all files necessary during the early boot process before
the root file system is mounted. It is important that your [initramfs] is
configured correctly to ensure the system can complete the boot process. In
Arch Linux, [mkinitcpio] is the default way to create the [initramfs]. In our
case, we need to ensure that the appropriate [hooks] are set in
`/etc/mkinitcpio.conf`:

-   [`lvm2` and `encrypt` hooks] ([mkinitcpio example])
-   [`resume` hook]
-   [`microcode` hook]
-   `keyboard` and `keymap` must be before `encrypt` for typing your password
-   `btrfs` hook if your [Btrfs] file system is on multiple devices

[Initramfs]: https://wiki.archlinux.org/title/Arch_boot_process#initramfs
[mkinitcpio]: https://wiki.archlinux.org/title/Mkinitcpio
[hooks]: https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks
[`lvm2` and `encrypt` hooks]: https://wiki.archlinux.org/title/Dm-crypt/System_configuration#mkinitcpio
[mkinitcpio example]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_3
[`resume` hook]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs
[`microcode` hook]: https://wiki.archlinux.org/title/Microcode#mkinitcpio

Be sure to put the hooks in the correct order.

Note that you could choose to use the systemd-based setup instead of the
default Busybox-based setup. This basically means using a different set of
hooks which have various differences. In particular, the [`sd-encrypt`] hook
supports some additional features that the Busybox equivalent `encrypt` hook
does not. If you need these additional features, feel free to switch over, but
otherwise, the default Busybox-based setup is sufficient for my needs.

[`sd-encrypt`]: https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Using_encrypt_hook

#### Unified kernel image

Next, we will configure [mkinitcpio] to build a [unified kernel image]. This
packages the kernel and everything it needs into a single executable which will
ensure that all relevant files can be signed in one go. Otherwise, it will be
possible to manipulate other unsigned files without being detected by [Secure
Boot]. Consult the [mkinitcpio instructions] and make sure to do the following:

1.  Add `/etc/cmdline.d/*.conf` files to set the [kernel parameters]:
    -   [`cryptdevice=UUID=XXX:cryptlvm`]
    -   `root=/dev/MainVolGrp/btrfs`
    -   [`rootflags=subvol=@root`]
    -   [`resume=/dev/MainVolGrp/swap`]
    -   Do *not* set `initrd` since `mkinitcpio` will add it for us
2.  Update the `/etc/mkinitcpio.d/linux.preset` file
3.  Ensure `sbctl` is installed so it automatically signs the [unified kernel
    image] whenever `mkinitcpio` is run

[unified kernel image]: https://wiki.archlinux.org/title/Unified_kernel_image
[mkinitcpio instructions]: https://wiki.archlinux.org/title/Unified_kernel_image#mkinitcpio
[kernel parameters]: https://wiki.archlinux.org/title/Kernel_parameters
[`cryptdevice=UUID=XXX:cryptlvm`]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader_2
[`rootflags=subvol=@root`]: https://wiki.archlinux.org/title/Btrfs#Mounting_subvolume_as_root
[`resume=/dev/MainVolGrp/swap`]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Pass_hibernate_location_to_initramfs

### rEFInd Boot Manager

I used to recommend using [GRUB], but even though it's the default for a lot of
distros, it's a bit annoying to configure and likely has more features than you
need. In the old guide, I used it because it was the only boot manager that
could read from both a [dm-crypt] encrypted partition and the [Btrfs] file
system. Now that we aren't using an encrypted `/boot`, we can use my preferred
boot manager, [rEFInd]. It's a lot simpler since it almost automatically works
out-of-the-box, even in surprisingly complex situations like dual-booting with
Windows or macOS. There are also many [rEFInd themes] (my favorite is
[refind-theme-regular]).

[GRUB]: https://wiki.archlinux.org/title/GRUB
[rEFInd]: https://wiki.archlinux.org/title/REFInd
[rEFInd themes]: https://www.rodsbooks.com/refind/themes.html
[refind-theme-regular]: https://github.com/bobafetthotmail/refind-theme-regular

Follow the instructions for [Secure Boot with rEFInd]. Very conveniently,
[rEFInd] has built-in support to automatically sign itself with your MOK using
`sbsign`. You can either let [rEFInd] create its own keys and import them into
`sbctl`, or you can copy your `sbctl` generated keys to `/etc/refind.d/keys/`
for [rEFInd] to use. Note that you may need to set up an icon yourself since
rEFInd can't automatically discern one from a [unified kernel image].

[Secure Boot with rEFInd]: https://wiki.archlinux.org/title/REFInd#Secure_Boot

### Rebuild initramfs

Don't forget to run `mkinitcpio -P` to rebuild your [initramfs] after you are
done configuring [mkinitcpio]. Forgetting to do so likely means your system
won't boot since it'll use the old [initramfs] that's not configured/signed
correctly. Assuming you registered/generated your MOK with `sbctl`, you can use
`sbctl verify` to confirm that the [unified kernel image] was signed properly.

### Enroll MOK

As described earlier, you will need to add any MOKs you used for signing EFI
binaries to the [shim]. You can use `mokutil` to do this. Note that `mokutil`
only buffers the changes for the next reboot since it can't make any changes
until then. If you prefer, you can usually do the same actions by booting
MokManager directly in [UEFI], but I find `mokutil` is more convenient.

Depending on whether you relied on [rEFInd] or [sbctl] to generate your MOK,
they will probably be in one of these locations:

-   `/etc/refind.d/keys/`
-   `/var/lib/sbctl/keys/`

If you did generate the key using `sbctl`, you'll probably need to convert it
to the correct format for `mokutil` with:

```sh
$ openssl x509 -outform der -in /var/lib/sbctl/keys/db/db.pem -out /tmp/sbctl_db.crt
```

When you boot, MokManager will pop up if [shim] is unable to execute your boot
loader, in which case, you need to use the MokManager's UI to find the key and
add it. After this, the [shim] won't need to invoke MokManager for your boot
loader, but you may still have problems if your kernel wasn't signed properly
or its MOK wasn't added.

### Reboot

Now that everything is set up correctly (supposedly), you should be able to
exit the chroot environment, `umount -R /mnt` everything, and simply `reboot`
your machine. If your system boots up, great you did it! If it didn't, well,
that's just how it goes sometimes, and you'll have to go troubleshoot what
exactly went wrong.

## Post-installation
