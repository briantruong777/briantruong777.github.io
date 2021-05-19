---
---

I've been a big fan of [Arch Linux] for a long time now. Though it is a pain to
set up, it offers a great opportunity to learn exactly how everything fits
together in a Linux distro. There's also that pleasant satisfaction when
everything is finally set up exactly the way I like it. In the interest of
making this process easier, I've written down my personal Arch Linux setup
here. Though this guide does set up Arch as a [VirtualBox] guest, a lot of this
applies to my other setups as well.

[Arch Linux]: https://archlinux.org/
[VirtualBox]: https://www.virtualbox.org/

## Overview

In general I'll also explain why I made certain choices rather than just going
through all the steps. After all, the benefit of Arch is that you get to make
the choice, and I would hate to completely take that away from you.

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
