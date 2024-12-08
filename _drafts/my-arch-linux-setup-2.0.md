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

Compared to the original post, the many differences including my reduced
patience for customization, unencrypted `/boot`, no mention of virtualization,
and additional laptop guidance. Here's an overview of the setup:

-   [UEFI] with [Secure Boot] enabled
-   [rEFInd] boot manager
-   Partitioning ([GPT])
    -   [EFI system partition] mounted to `/boot`
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
[Secure Boot]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot
[rEFInd]: https://wiki.archlinux.org/title/REFInd
[GPT]: https://wiki.archlinux.org/title/Partitioning#GUID_Partition_Table
[EFI system partition]: https://wiki.archlinux.org/title/EFI_system_partition
[SystemRescue]: https://www.system-rescue.org
[dm-crypt]: https://wiki.archlinux.org/title/Dm-crypt
[LVM]: https://wiki.archlinux.org/title/LVM
[Btrfs]: https://wiki.archlinux.org/title/Btrfs
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
    -   From this point on, you may want to use [tmux] to allow scrolling back to previous output
3.  Set the keyboard layout if needed
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
which is now stored in the [EFI system partition]. This prevents synchronously
snapshotting the kernel with the rest of the root file system, but since the
original guide was written, I've never had a need to atomically rollback the
kernel and kernel modules together. Thus, it doesn't seem worth the complexity
and suffering from being restricted to [GRUB]. Plus, I am now going to use
[Secure Boot], so the other benefit of preventing tampering of the kernel is
already covered.

[GRUB]: https://wiki.archlinux.org/title/GRUB

And yes, you heard that right, I'm using [Secure Boot]. I used to think it was
overkill unless some nation's intelligence agency is gunning for you, but
malware that infects your boot loader or kernel is not as rare as you might
think. Plus, it is supported by both Ubuntu and Fedora which makes it
commonplace even for Linux machines.

The other major difference from the original guide is that I use [LVM] to
create separate logical volumes for the [Swap] space and [Btrfs] file system. I
used to suggest using a Swap file instead, but it's a bit finicky to set up on
[Btrfs], and [LVM] provides the equivalent convenience of being able to resize
it on-the-fly. Plus, having a single [LVM] partition means we can
encrypt/decrypt the Swap and root file system in one fell swoop. Finally, [LVM]
gives much more flexibility in the future for partitioning shenanigans.

[Swap]: https://wiki.archlinux.org/title/Swap

A minor difference from the original guide is that I use [Snapper] to
automatically take [Btrfs] snapshots, so the layout of [Btrfs] subvolumes is a
bit different. Note that if you want convenient rollback, you may want to
follow the [suggested subvolume layout] or even set up [GRUB] to allow choosing
a snapshot to boot into. I don't bother since I've never needed to do a full
rollback like that (Arch Linux is surprisingly reliable for me), and if the
need arises, I don't mind doing it manually (that's what the [SystemRescue]
toolkit is for).

[suggested subvolume layout]: https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout

#### Create partitions

We will only need 2 partitions since the primary partition will have [LVM]
setup. Be sure to use the [GPT] partitioning scheme which is the standard
nowadays despite `fdisk` defaulting to MBR (or just use `gdisk`).

[GPT]: https://wiki.archlinux.org/title/Partitioning#GUID_Partition_Table

1.  [EFI system partition] (2 GiB)
2.  Primary partition (remaining space)

All [UEFI] systems require an [EFI system partition]. In our case, we are going
to be mounting this partition as `/boot` on our system meaning we will store
our kernel there (and later on [SystemRescue] installation). This means we need
a decent amount of space in there, so I'd recommend making the partition at
least 2 GiB. You could go even further if you are paranoid. Don't forget to
format the EFI partition as FAT32.

#### Encryption

For our primary partition, we first need to encrypt it using [dm-crypt]. There
are [many possible encryption setups], but the one we will use is [LVM on LUKS]
for the reasons described earlier.

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

#### LVM

Now create an [LVM] physical volume on the `/dev/mapper/` entry for the
encrypted partition, and put it in a volume group. Then create two logical
volumes in the volume group:

1.  [Swap] logical volume
2.  [Btrfs] logical volume

Set up and enable [Swap] the appropriate logical volume. [Btrfs] will be a bit
more complicated.

#### Btrfs

In case you are not aware of it, [Btrfs] is a CoW (copy-on-write) file system
that has a wide variety of features including utilizing multiple devices,
efficient snapshotting, and transparent compression. These features do come at
a performance cost, and some of them are downright broken like [RAID5/6], but
at least for my purposes, the good outweighs the bad.

[RAID5/6]: https://btrfs.readthedocs.io/en/stable/btrfs-man5.html#raid56-status-and-recommended-practices

One big weakness of [Btrfs] is that it behaves poorly when out of disk space
since even deletions require writing new bytes. Such situations are more likely
than one might expect thanks to the convenience of snapshots.

Another weakness is the poor performance for workloads that need to update
existing blocks repeatedly like databases and VMs. Thankfully, [Btrfs] supports
[disabling CoW] for specified directories/files which mitigates the issue.

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

|                        | Path                           | Mounted At               |
| [dm-crypt]             | `/dev/nvme0n1p2`               | `/dev/mapper/cryptlvm`   |
| [LVM]                  | `/dev/mapper/cryptlvm`         | `/dev/mapper/MainVolGrp` |
| [Swap]                 | `/dev/mapper/MainVolGrp/swap`  | Enabled using `swapon`   |
| [Btrfs] `@root`        | `/dev/mapper/MainVolGrp/btrfs` | `/`                      |
| [Btrfs] `@home`        | `/dev/mapper/MainVolGrp/btrfs` | `/home`                  |
| [EFI system partition] | `/dev/nvme0n1p1`               | `/boot`                  |

## Configure the system
## Post-installation
