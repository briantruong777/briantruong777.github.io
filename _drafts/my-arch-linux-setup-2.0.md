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

Directly from the [installation guide]:

1.  [Download] the installation image and install onto a [USB drive]
2.  Boot the live environment from the USB drive
    -   You may need to disable [Secure Boot] first
    -   From this point on, you may want to use [tmux] to allow scrolling back to previous output
3.  Set the keyboard layout (and font) if needed
4.  Connect to the internet
5.  Update the system clock

[Download]: https://archlinux.org/download
[USB drive]: https://wiki.archlinux.org/title/USB_flash_installation_medium
[tmux]: https://wiki.archlinux.org/title/Tmux

### Partitioning

Your choice of partitioning scheme and file system are the first of the many
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
malware that infects your boot loader or even BIOS is not as rare as you might
think. Plus, it is supported by both Ubuntu and Fedora which makes it
commonplace even for Linux machines.

The other major difference from the original guide is that I use [LVM] to
create separate logical volumes for the [swap] space and [Btrfs] file system. I
used to suggest using a swap file instead, but it's a bit finicky to set up on
[Btrfs], and [LVM] provides the equivalent convenience of being able to resize
it on-the-fly. Plus, having a single [LVM] partition means we can
encrypt/decrypt the swap and root file system in one fell swoop. Finally, [LVM]
gives much more flexibility in the future for partitioning shenanigans.

[swap]: https://wiki.archlinux.org/title/Swap

A minor difference from the original guide is that I use [Snapper] to
automatically take [Btrfs] snapshots, so the layout of [Btrfs] subvolumes is a
bit different. Note that if you want convenient rollback, you may want to
follow the [suggested subvolume layout] or even set up [GRUB] to allow choosing
a snapshot to boot into. I don't bother since I've never needed to do a full
rollback like that (Arch Linux is surprisingly reliable for me), and if the
need arises, I don't mind doing it manually (that's what the [SystemRescue]
toolkit is for).

[suggested subvolume layout]: https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout

#### EFI system partition
#### Encrypted partition
#### LVM
#### Btrfs

## Installation
## Configure the system
## Post-installation
