---
---

This is the next version of my [original guide] to setting up [Arch Linux]. Why
am I writing this? Well, I'm getting a new [Framework] Laptop 13, and I want to
install Arch Linux on it. Yup, that's pretty much it. Some of my preferences
have changed, but don't be surprised if much of this feels the same.

[original guide]: {% post_url 2022-01-17-my-arch-linux-setup %}
[Arch Linux]: https://archlinux.org
[Framework]: https://frame.work

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
    -   [Uncomplicated Firewall]
    -   [systemd-timesyncd]
    -   [restic] and [resticprofile] for backups
    -   [Paru] for installing from the [AUR]
    -   [GNOME] desktop environment
    -   [fish] (friendly interactive shell)

[UEFI]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface
[Hibernation]: https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation
[NetworkManager]: https://wiki.archlinux.org/title/NetworkManager

## Pre-installation

The initial steps from the [installation guide]:

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

Set up and enable [Swap] the appropriate logical volume. Technically speaking,
your [Swap] should be at least as big as your total RAM so that [hibernation]
is guaranteed to successfully store all the memory. In practice however,
compression does give a decent chance of success even with less space, so feel
free to use less space.

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

-   `@root` mounted at `/mnt`
-   `@home` mounted at `/mnt/home`

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
start you off, here's the packages I like to install, but obviously you can
omit them or choose your own equivalents:

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
    -   [NetworkManager], an all-in-one network management program
-   `base-devel`
    -   Tools needed for compiling packages
-   `man-db` and `man-pages`
    -   man pages
-   `tmux`
    -   [Tmux] the terminal multiplexer
    -   Enables multiple terminals before you have a desktop environment
-   `neovim`
    -   Fork of `vim` that has better defaults
-   `fish` (also `python` for automatic parsing of man pages)
    -   [fish], the friendly interactive shell with nice default behavior
-   `git`
    -   The gold standard for version control of open source code
    -   I pretty much always end up needing this for something
-   `fd`
    -   Alternative to `find` for searching files/directories by name
    -   User friendly, fast, obeys `.gitignore`, colors, etc.
-   `ripgrep`
    -   Alternative to `grep` that is faster and defaults to recursive search
    -   User friendly, fast, obeys `.gitignore`, colors, etc.
-   `bat`
    -   Alternative to `cat` for displaying text files for human consumption
    -   Color syntax highlighting, automatic pager, `git` diff, etc.

## Configure the system

And now the fun part where you have to configure everything correctly otherwise
your system won't boot properly and you will have no clue what the heck went
wrong so you desperately try rerunning all sorts of commands until you are
forced to confront the fact that you have no option left except to delete
everything and start over from the beginning. But anyway, I'm sure that can't
possibly happen to you, so no need to worry. Probably.

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
    Supposedly the [Framework Laptop 13] (AMD 7040) supports Secure Boot
    [without any Microsoft/vendor keys], but it may prevent applying [future
    firmware updates].

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

Installing `sbctl` also adds [pacman] and [mkinitcpio] hooks that will sign
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
[mkinitcpio] and [pacman] hooks, so I really only install `sbsigntools` for the
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
$ curl -L -O https://aur.archlinux.org/cgit/aur.git/snapshot/shim-signed.tar.gz
$ tar -xzf shim-signed.tar.gz
$ chmod 777 shim-signed
$ cd shim-signed
$ less *  # Confirm files look safe
$ runuser -u nobody makepkg
$ pacman -U shim-signed*.pkg.tar.*
```

[shim-signed]: https://aur.archlinux.org/packages/shim-signed/

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
    image] with its imported/generated keys whenever `mkinitcpio` is run

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

Depending on whether you relied on [rEFInd] or `sbctl` to generate your MOK,
the key will probably be in one of these locations:

-   `/etc/refind.d/keys/`
-   `/var/lib/sbctl/keys/`

If you did generate the key using `sbctl`, you'll probably need to convert it
to the correct format for `mokutil` with:

```sh
$ openssl x509 -outform der -in /var/lib/sbctl/keys/db/db.pem -out /tmp/sbctl_db.crt
```

After using `mokutil`, when you boot, MokManager will prompt you to confirm you
meant to add the keys.

If for whatever reason the [shim] is unable to execute your boot loader,
MokManager will also pop up and let you pick a binary/key stored in the [EFI
system partition] to add to its database. After this, the [shim] won't need to
invoke MokManager for your boot loader, but you may still have problems if your
kernel wasn't signed properly or its MOK wasn't added.

### Reboot

Now that everything is set up correctly (supposedly), you should be able to
exit the chroot environment, `umount -R /mnt` everything, and simply `reboot`
your machine. If your system boots up, great you did it! If it didn't, well,
that's just how it goes sometimes, and you'll have to go troubleshoot what
exactly went wrong.

## Post-installation

At this point, your system should be capable of booting on its own, connecting
to the internet, and installing any package you like, so feel free to close
this guide and do whatever you want. The rest of this guide is effectively just
a list of software and other [general recommendations] that I like to follow.

[general recommendations]: https://wiki.archlinux.org/title/General_recommendations

### User account

Currently the only user you can log in as is the root user which is not a good
user to use regularly. You either run the risk of accidentally breaking your
system or someone hacking your account and now having full access to your
entire system. See [users and groups] documentation on how to create your own
user account. Also consider adding your user to the `wheel` group which
conventionally has access to administrate the system and use [sudo].

[users and groups]: https://wiki.archlinux.org/title/Users_and_groups

As you might've guessed, I recommend installing [sudo] so your new user account
can actually administrate the system as needed. Once this is done and you've
tested it works, you can now [disable root login] to prevent anyone from
logging in directly as the root user.

[sudo]: https://wiki.archlinux.org/title/Sudo
[disable root login]: https://wiki.archlinux.org/title/Sudo#Disable_root_login

On a separate note, it's also nice to [disable the password timeout] for [sudo]
in case you have a long running script (e.g., [AUR] helper) that needs [sudo]
access at the end. This lets you come back to your computer and type in your
password to continue the without failing the script.

```
Defaults passwd_timeout=0
```

[disable the password timeout]: https://wiki.archlinux.org/title/Sudo#Disable_password_prompt_timeout

You may want to install [polkit] which is a framework for allowing unprivileged
users to execute certain commands without giving full root access like [sudo]
does. In particular, it allows you to run `poweroff` and `reboot` without
needing [sudo]. [Polkit] also considers the `wheel` group as an administrative
group.

[polkit]: https://wiki.archlinux.org/title/Polkit

Finally, if you want the standard set of directories for Linux (`Downloads/`,
`Desktop/`, `Music/`, etc.), you should take a look at [XDG user directories].

[XDG user directories]: https://wiki.archlinux.org/title/XDG_user_directories

### Installing packages

In the unlikely event you've somehow gotten this far without understanding what
you've been doing, [pacman] is Arch Linux's package manager which allows you to
install lots of software from the official repository. Assuming you have an
internet connection, [pacman] should just work out-of-the-box, but there are a
few settings in `/etc/pacman.conf` you may want to set.

-   `Color` to enable color output in the terminal
-   `ParallelDownloads` to speed up downloading lots of packages

[pacman]: https://wiki.archlinux.org/title/Pacman

Though the Arch Linux installation environment does this already, you may want
to use [reflector] to build an optimal list of repository mirrors. Depending on
how you configure it, [reflector] will test the freshness and download speeds
of all the Arch Linux repository mirrors and select the best ones to store into
`/etc/pacmand.d/mirrorlist`. You could run this regularly, but I suggest
configuring [reflector] and running it once when you first install, and then
again whenever you travel or move to a new location.

[reflector]: https://wiki.archlinux.org/title/Reflector

#### AUR

Though the official repository has most of the software you might want, there
will always be some programs or variants that aren't available which is where
the [AUR] (Arch User Repository) comes in. The [AUR] is where anyone can upload
packages (`PKGBUILD` and other needed files) which you can then download in
order to build the package locally. This usually involves compiling the program
from scratch, but some [AUR] packages just download an existing compiled binary
and just rearrange it for installation.

[AUR]: https://wiki.archlinux.org/title/Arch_User_Repository

Now, before we get any further, let me repeat that. *Anyone* can upload
packages to the [AUR]. Though a popular package is less likely to have anything
harmful, there is no vetting process at all for any of these packages meaning
it could just be malware. Even if you confirmed a package is safe now, a future
update may introduce malicious code, so best practice is to inspect the diff
before installing the update.

To make this easier, there are a variety of [AUR helpers] which make it
convenient to install and update packages from the [AUR]. I personally use
[paru], but as with any of the [AUR helpers], the convenience encourages
complacency. You are responsible for the software you install on your system,
so do your research.

[AUR helpers]: https://wiki.archlinux.org/title/AUR_helpers
[paru]: https://github.com/Morganamilo/paru

For any of the [AUR helpers], you will still need to install them manually the
first time. Afterwards, they can usually update themselves, but it's still good
to know how to install [AUR] packages manually for when it occasionally fails.

```sh
$ cd /tmp
$ curl -L -O https://aur.archlinux.org/cgit/aur.git/snapshot/paru.tar.gz
$ tar -xzf paru.tar.gz
$ cd paru
$ less *  # Confirm files look safe
$ makepkg -si
```

### Desktop environment

Unless you are a masochist or allergic to GUIs, you probably want to install a
[desktop environment]. There are many to choose from, and you could even
install a plain [window manager]. I used to like [i3], but it was kind of a
pain compared to something that just works out-of-the-box. Plus, I use [GNOME]
and macOS at work, so it's just easier to go with [GNOME] for my personal setup
too. I'm not going to bother explaining how to use [GNOME] since it's pretty
simple.

[desktop environment]: https://wiki.archlinux.org/title/Desktop_environment
[window manager]: https://wiki.archlinux.org/title/Window_manager
[i3]: https://wiki.archlinux.org/title/I3
[GNOME]: https://wiki.archlinux.org/title/GNOME

Here are the desktop environments and window managers you may want to consider:

-   [GNOME]
    -   Popular all-in-one desktop environment that is macOS-like
-   [KDE Plasma]
    -   The other popular all-in-one desktop environment that takes a
        Windows-like approach instead
-   [Xfce]
    -   Lightweight desktop environment based on GTK
-   [Cinnamon]
    -   Fork of [GNOME] that maintained the more classic desktop paradigm
-   [i3]
    -   Tiling window manager with a binary tree paradigm and support for
        floating windows
-   [Sway]
    -   Drop-in replacement for [i3] that supports Wayland
-   [Hyprland]
    -   Tiling window manager that focuses on looking pretty with nice
        animations

[KDE Plasma]: https://wiki.archlinux.org/title/KDE
[Xfce]: https://wiki.archlinux.org/title/Xfce
[Cinnamon]: https://wiki.archlinux.org/title/Cinnamon
[Sway]: https://wiki.archlinux.org/title/Sway
[Hyprland]: https://wiki.archlinux.org/title/Hyprland

If you go with a [desktop environment], it'll probably contain everything you
need, but if you go with a [window manager], you'll have to add in more things
like a display manager, status bar, screen lockers, notification daemon,
application launcher, terminal emulator, audio system, etc.

### Terminal and shell

If you installed a [desktop environment], it probably comes with a default
terminal emulator that should satisfy most people's needs. I personally have
used [GNOME Terminal] extensively, and it's had all the features I've needed. I
usually just rely on [GNOME]'s basic window management to deal with multiple
terminals, and [tmux] when over ssh.

[GNOME Terminal]: https://help.gnome.org/users/gnome-terminal/stable/
[tmux]: https://wiki.archlinux.org/title/Tmux

However, some other terminals you may want to consider are [Alacritty] and
[kitty]. They both use GPU rendering to speed things up (not that you need 144
fps for your terminal text output) and have up-to-date, modern sensibilities.
[kitty] in particular is interesting because it can load the scrollback buffer
or the output of the last command into your pager of choice which is great
since usually I'd just rerun the command again and accept the shame from my
inability to plan ahead. [kitty] annoyingly does not support using the mouse to
click-and-drag the scrollbar, but it's other features like jumping to
previous/next prompt make up for it.

[Alacritty]: https://wiki.archlinux.org/title/Alacritty
[kitty]: https://wiki.archlinux.org/title/Kitty

If you want a terminal that uses GPU rendering to slow things down instead,
there's [cool-retro-term] which makes your terminal look cool and retro.
Definitely recommend toying around with the settings, though you may cause some
eye damage if you take things too far.

[cool-retro-term]: https://github.com/Swordfish90/cool-retro-term

For my shell, I prefer [fish] nowadays since it has lots of nice features
(advanced autocompletion, command highlighting, right-hand prompts) and most of
these just work with basically no configuration. For better or worse, [fish] is
not POSIX compliant so it is not compatible with `bash` syntax, scripts, and
conventions. Some of its choices are better, but differing from the norm can
make things difficult. If you find that to be unacceptable, I'd recommend [zsh]
which is POSIX compliant, but after a decent amount of configuration, can still
provide the same features.

[fish]: https://wiki.archlinux.org/title/Fish
[zsh]: https://wiki.archlinux.org/title/Zsh

### Fonts

[Fonts] are inherently subjective and there's plenty of choices out there, but
I'll list some below that are popular or have piqued my interest.

[Fonts]: https://wiki.archlinux.org/title/Fonts

-   [Noto]
    -   `noto-fonts`, `noto-fonts-cjk`, `noto-fonts-emoji`
    -   Designed to provide the maximal coverage possible for all of Unicode so
        there will be "no more tofu"
    -   Installing everything will take up a decent amount of space
-   [DejaVu]
    -   `ttf-dejavu`
    -   Classic font family based on Bitstream Vera designed for general usage
-   [Roboto]
    -   `ttf-roboto`, `ttf-roboto-mono`
    -   Developed by Google for Android
-   [Fira]
    -   `ttf-fira-sans`/`otf-fira-sans`, `ttf-fira-mono`/`otf-fira-mono`,
    -   Developed by Mozilla as a general usage font inspired by [FF Meta]
-   [IBM Plex]
    -   `ttf-ibm-plex`
    -   Developed by IBM as a modern font inspired by IBM's past

[Noto]: https://fonts.google.com/noto
[DejaVu]: https://dejavu-fonts.github.io/
[Roboto]: https://en.wikipedia.org/wiki/Roboto
[Fira]: https://mozilla.github.io/Fira
[FF Meta]: https://en.wikipedia.org/wiki/FF_Meta
[IBM Plex]: https://www.ibm.com/plex/

The previous fonts do have monospace variants, but the next fonts are
specifically aimed at programmers. Some have a [Nerd Fonts] variant which adds
lots of icons useful for programmers (though this is unnecessary for some
terminal emulators like [kitty]). To compare between them, try out
<https://www.programmingfonts.org>.

-   [Hack]
    -   `ttf-hack`/`ttf-hack-nerd`
    -   Basically [DejaVu] with some alterations for source code
-   [Fira Code]
    -   `ttf-fira-code`/`ttf-firacode-nerd`
    -   [Fira] Mono with some alterations for source code
    -   Fine-tuning to align punctuation and some letter pairs
-   [Monaspace]
    -   `otf-monaspace`/`otf-monaspace-nerd`/`ttf-monaspace-variable`
    -   Developed by GitHub for source code
    -   Features "texture healing" for better readability
    -   Contains multiple variants that are compatible with each other
-   [JetBrains Mono]
    -   `ttf-jetbrains-mono`/`ttf-jetbrains-mono-nerd`
    -   Developed by JetBrains for source code
    -   Increased letter height
    -   Ovals approximating rectangular symbols
    -   Leans towards simplicity over style (though not too much)
-   [Cascadia Code]
    -   `ttf-cascadia-code`/`ttf-cascadia-code-nerd`/`ttf-cascadia-mono-nerd`
    -   Developed by Microsoft for the new Windows Terminal
-   [Inconsolata]
    -   `ttf-inconsolata`/`ttf-inconsolata-nerd`
    -   Clone of Microsoft's Consolas
-   [Iosevka]
    -   `ttc-iosevka`/`ttf-iosevka-nerd`
    -   Slender slab-serif font designed to minimize horizontal space used
    -   Has lots of variants with different styles and other customizations
-   [Comic Mono]
    -   `ttf-comic-mono-git`
    -   Comic Sans clone but in monospace
    -   What further explanation could you need?
-   [Hermit]
    -   `otf-hermit`/`otf-hermit-nerd`
    -   Handcrafted, pragmatic font that is compact and IMO retro-futuristic

[Nerd Fonts]: https://www.nerdfonts.com/
[Hack]: https://sourcefoundry.org/hack
[Fira Code]: https://github.com/tonsky/FiraCode
[Monaspace]: https://monaspace.githubnext.com/
[JetBrains Mono]: https://www.jetbrains.com/lp/mono/
[Cascadia Code]: https://github.com/microsoft/cascadia-code
[Inconsolata]: https://levien.com/type/myfonts/inconsolata.html
[Iosevka]: https://typeof.net/Iosevka/
[Comic Mono]: https://dtinth.github.io/comic-mono-font/
[Hermit]: https://pcaro.es/hermit/

### Firewall

Firewalls are not exactly required especially on Linux where ports are closed
by default. However, if you end up running malicious software or misconfigure
something, you may expose a port to the outside world which could leave you
vulnerable.

Nowadays, I use [Uncomplicated Firewall] (`ufw`) for this. As the name implies,
it has simpler configuration than raw `iptables` or `nftables` usage and I
don't find myself missing any of the more advance stuff. If you are ssh'd into
your machine, be sure to add an exception for that first to avoid accidentally
locking yourself out.

[Uncomplicated Firewall]: https://wiki.archlinux.org/title/Uncomplicated_Firewall

### Time syncing

In general, you should ensure your system regularly synchronizes its time since
no quartz clock will remain accurate forever. [systemd-timesyncd] is a simple
SNTP client that can get the job done and is already included with your system.
However, if you need better accuracy, consider using [Chrony]. For personal
devices, I don't bother, but it could be more useful for servers where accurate
time for logging or even correctness of some software is critical.

[systemd-timesyncd]: https://wiki.archlinux.org/title/Systemd-timesyncd
[Chrony]: https://wiki.archlinux.org/title/Chrony

### Hardware

Depending on your hardware, there are various things you may need to configure.

#### Graphics cards

If you have a dedicated graphics card, take a look at the [NVIDIA] and [AMD
GPU] drivers. If you have Intel integrated graphics, check out [Intel
graphics]. AMD integrated graphics are already built into the Linux kernel.
Regardless of your setup, you probably want to look at [OpenGL] and [Vulkan] to
make sure you are utilizing your hardware to its maximum potential.

[NVIDIA]: https://wiki.archlinux.org/title/NVIDIA
[AMD GPU]: https://wiki.archlinux.org/title/AMDGPU
[Intel graphics]: https://wiki.archlinux.org/title/Intel_graphics
[OpenGL]: https://wiki.archlinux.org/title/OpenGL
[Vulkan]: https://wiki.archlinux.org/title/Vulkan

#### Sound

I'm no audiophile, but I would like for basic things like videos and music to
play properly. Assuming your [desktop environment] didn't set this up for you
already, you'll have to install a [sound server] to start with. I'm not going
to claim I have even the faintest idea of the differences between all the
solutions, but [PipeWire] works perfectly fine for my purposes and is well
integrated into [GNOME]. If you install the right packages, it can also
integrate with JACK and PulseAudio clients, so assuming those integrations work
for you, it seems like a no-brainer to me.

[sound server]: https://wiki.archlinux.org/title/Sound_system
[PipeWire]:  https://wiki.archlinux.org/title/PipeWire

#### Bluetooth

I really only use [Bluetooth] to connect my headphones, but even then, it can
be quite finicky at times. Be sure to follow the instructions for [Bluetooth]
carefully.

[Bluetooth]: https://wiki.archlinux.org/title/Bluetooth

#### Laptops

With laptops, power usage and battery life are incredibly important, and
unfortunately Linux is pretty weak in this department. If you are willing to
put in excessive amounts of effort, you can certainly cut out all the fluff and
run a lean machine that is very inconvenient to use, but even comparing that to
any modern macOS or Windows machine with its out-of-the-box configuration will
be a struggle. Maybe you are reading this far enough in the future for this
efficiency gap to be closed, but I'm not very hopeful personally.

Anyway, check out all the [Laptop] specific guidance which covers all sorts of
things. Assuming you followed this guide correctly, we already set up
[hibernation], but check out the rest of the [Power management] guidance for
more power saving techniques.

[Laptop]: https://wiki.archlinux.org/title/Laptop
[Power management]: https://wiki.archlinux.org/title/Power_management

In addition to the above general guidance, you should also check if there is an
Arch Wiki page specific to your computer/laptop. For me, there is a page
dedicated to the [Framework Laptop 13]. In fact, the next time you buy a
computer, consider checking for a dedicated Arch Wiki page first to see what
you can expect to work or not work.

[Framework Laptop 13]: https://wiki.archlinux.org/title/Framework_Laptop_13

### Backups

There are only two types of people: those who have backups and those who
haven't learned why they should have backups. The long story short is that
nothing is 100% reliable, so if you have data that would be painful to lose,
you should have backups of it. There are many possible backup schemes with
varying pros and cons. To start with, I like to classify the types of copies.

1.  Convenience copies
    -   Primary copies that you use regularly or plan on using in the future
    -   For making your life easier rather than for redundancy
2.  Historical copies
    -   Copies of older or deleted versions of the data
    -   For undoing mistakes
3.  Backup copies
    -   Copies in different physical location and digital format
    -   For when your house burns down or you get locked out of your account

Having a distinct copy for each category is a good starting point. Be wary of
copies that are linked together in some way since they may have shared fate.
For example, if you get locked out of your Google account, it may also disable
your Android phone, so those are effectively a single copy. Also consider if
multiple copies often share the same location, e.g., your laptop usually stays
at home and then your house burns down. Finally, be sure you understand what
having different digital formats means. Using the same software for all copies
means a bug in that software might corrupt all the copies. Ideally your backup
copies would use different software, i.e., a different digital format.

Since our system is on [Btrfs], I like to use [Snapper] to make historical
copies in case I need to undo a system update or accidental overwrite/deletion
of my personal files. Then I use [restic] to store backups on my "home server"
in a different format. This makes it a different physical location (not in my
backpack) and different digital format (not on [Btrfs]). I also upload an
encrypted copy to the cloud every now and then, though I consider this less
important since I've never had my house burn down (famous last words).

#### Snapper

[Snapper] is a tool that can automatically create and clean up [Btrfs]
snapshots on a schedule. It also has pre/post snapshots which are intended to
capture the state before and after some change. It is intended for openSUSE so
some of its features (e.g. rollback) aren't integrated very well in other
contexts, but it does have some more advanced features like reducing the number
of snapshots when space is running low. One integration that does work is
installing `snap-pac` which will add [pacman] hooks to take pre and post
snapshots automatically whenever you install/update/remove packages.

[Snapper]: https://wiki.archlinux.org/title/Snapper

#### restic

[restic] is a tool for creating and managing backups. Though it is somewhat
low-level and difficult to set up, it has lots of nice features like cross-OS
compatibility, de-duplication, compression, encryption, remote/cloud storage
support, etc. The main thing missing is built-in support for scheduling
automatic backups (kind of a big gap but whatever). Thankfully, that's where
[resticprofile] comes in. There are a variety of tools that can automatically
schedule things, but [resticprofile] works on Windows and Linux and it is
configured via a TOML file which works for me.

[restic]: https://wiki.archlinux.org/title/Restic
[resticprofile]: https://creativeprojects.github.io/resticprofile

### SystemRescue

Hopefully you are already aware of this, but [Arch Linux] is not considered a
stable system due to its rolling release nature. In practice, I've rarely had
it break on me, but when it does happen, it can be really painful if it causes
the system to fail to boot. That's where having the [SystemRescue] toolkit
installed is handy since you can boot into that and fix the main system. See
the [SystemRescue install instructions]. You'll probably need to configure
[rEFInd] to correctly find the kernel (`vmlinuz`) and pass in the required
[kernel parameters].

[SystemRescue]: https://www.system-rescue.org
[SystemRescue install instructions]: https://www.system-rescue.org/manual/Installing_SystemRescue_on_the_disk/

Since you probably won't ever bother with updating this installation, you could
just approve the hash of the binary in the [shim]. Otherwise, you can use
`sbctl` to sign the `vmlinuz` file with the same MOK you used for the main
system.

## The End

Alright, you've reached the end of this guide, congratulations! The old guide
had some guidance on [system maintenance], but I don't think I'll bother since
you either already know what to expect from [Arch Linux], or you deserve a rude
awakening for trying to use something without understanding the consequences.
Enjoy your new [Arch Linux] system!

[system maintenance]: https://wiki.archlinux.org/title/System_maintenance
