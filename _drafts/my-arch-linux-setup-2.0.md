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

Compared to the original post, the main differences are my reduced patience for
customization and a simpler encryption setup. There will also be more laptop
specific guidance since this is intended for my new Framework laptop. Here's an
overview of the setup:

-   [UEFI] with [Secure Boot] enabled
-   [rEFInd] boot manager
-   Partitioning ([GPT])
    -   [EFI system partition] mounted to `/boot`
    -   [dm-crypt] encrypted [LVM] partition
        -   `swap` logical volume
        -   `btrfs` logical volume
            -   `@root` mounted to `/`
                -   [Snapper] snapshots in `/.snapshots`
            -   `@home` mounted to `/home`
                -   [Snapper] snapshots in `/home/.snapshots`
-   [Hibernation] (suspend-to-disk)
-   Programs
    -   [NetworkManager]
    -   [Ufw] (Uncomplicated Firewall)
    -   [systemd-timesyncd]
    -   [Restic] and [resticprofile] for backups
    -   [Paru] the [AUR] helper
    -   [GNOME] desktop environment
    -   [fish] the friendly interactive shell

[UEFI]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface
[Secure Boot]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot
[rEFInd]: https://wiki.archlinux.org/title/REFInd
[GPT]: https://wiki.archlinux.org/title/Partitioning#GUID_Partition_Table
[EFI system partition]: https://wiki.archlinux.org/title/EFI_system_partition
[dm-crypt]: https://wiki.archlinux.org/title/Dm-crypt
[LVM]: https://wiki.archlinux.org/title/LVM
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
