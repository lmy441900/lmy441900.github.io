---
layout: post
title:  "Prevent Windows From Modifying UEFI Boot Sequence"
date:   2018-09-13
categories: en
---

**TL;DR: Set `Windows Boot Manager` entry to inactive.**

Perhaps you have been wondering why [this ArchWiki guide][guide] does not work. Everytime you set a correct boot sequence using `efibootmgr`, or removing `Windows Boot Manager` from UEFI boot table, Windows overrides these when it is booted. Even worse, my HP Pavilion g6-2328tx laptop will restore the boot order to its default by the UEFI itself. This HP even display this Windows boot manager entry as "OS System Administrator". This is just so annoying.

[guide]: https://wiki.archlinux.org/index.php/UEFI#Windows_changes_boot_order

My case is even worse: I have encrypted my Windows using [VeraCrypt], with the Windows boot loader keeps staying in the ESP (unencrypted). So everytime my laptop boot the Windows boot loader, it simply prompts "your system is damaged and need a repair" with a frightening blue background. Hence, I need a way to get rid of this, instead of stupidly pressing F9 (on my Dell laptop this is F12) everytime I power on a computer.

[VeraCrypt]: https://veracrypt.fr

One interesting fact I discovered during my dull reboot testing is that (I guess, perhaps) Windows checks the existence of `Windows Boot Manager` entry in the UEFI boot table. If there is none, Windows will set one, and modify the boot sequence so that `Windows Boot Manager` always gets loaded the first. If there is one, Windows will do nothing. So I let Windows to set the entry anyway, and reboot to Linux, run:

```bash
sudo efibootmgr --bootnum 0 --inactive
sudo efibootmgr -b 0 -A # shorthand
```

Where `0` is the boot entry number of `Windows Boot Manager`, in my case this is 0. Then, the entry will stay in the boot entry table, but will not get selected by the UEFI. In this way, Windows will do nothing to the boot order or boot entry. My HP Pavilion g6 laptop will not display a "system administrator" entry in the UEFI boot menu, and will not change the boot order[^1].

I think I should add this to ArchWiki?

## Notes

[^1]: Actually I've done this several times, and the firmware just kept reverting my changes. I suspect that when I did `efibootmgr -O` (which is a shorthand to `--delete-bootorder`), some weird (those "Internal Hard Disk") entries were cleared, and the firmware thought the UEFI boot sequence was damaged, so it regenerated it. I need more trials to figure out how on earth can I trigger this "repair".
