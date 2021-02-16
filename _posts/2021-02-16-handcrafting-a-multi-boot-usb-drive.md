---
layout: post
title:  "Handcrafting A Multi-boot USB Drive"
date:   2021-02-16
---

Building a multi-purpose USB thumb drive is like forging a swiss knife that suits one's need when in unusual situations, e.g. maintaining computers, recovering bootloader, rescuing data. In this blog post, I'd like to describe my recent design implemented on my USB drive.

I don't really trust those automatic, "one-click scripts" that magically make something for me; even if they have their source code available, reading and understanding the exact behaviors of them costs me tons of energy. So, on the one hand, I can't be bothered to use the automatic scripts, but on the another hand, I invest a lot of time crafting my own solutions. What a paranoid I am.

TL;BYHTR (Too Long, But You Have To Read). Sorry.

## Design Objectives

The USB drive is designed to be an everyday data drive as well as a bootable one. Partitions are divided using [Master Boot Record (MBR)][mbr], with [GNU GRUB 2][grub] being installed as the boot loader, and several [Live systems][live] (which lose information upon reboot) being installed, consisting the swiss knife.

In my previous design, the USB drive was partitioned with [GUID Partition Table (GPT)][gpt] only. It's reasonable to drop the support for PC BIOS based computers, since in 2021 we can seldom see one. Unfortunately, my ThinkPad X200 has a broken [TianoCore][edk] on [Coreboot][cb] (which was _also handcrafted; I'll write about this later_), but a working SeaBIOS and a working GRUB2. To boot external operating systems, I sometimes need to use a PC BIOS supported disk, which must be in the MBR format. Otherwise, GPT can be used instead.

One might also doubt why a Hybrid MBR cannot be made. The reason is simply because Hybrid MBR is chaotic and should not be considered at all.[^1] Even if a Hybrid MBR is deployed, there exists a great risk that different software and platforms may interpret the disk differently, causing unpredictable behaviors.

Overall, this time I tried to make it usable on more legacy platforms, so there must be compromises.

[mbr]: https://en.wikipedia.org/wiki/Master_boot_record
[grub]: https://www.gnu.org/software/grub/
[live]: https://en.wikipedia.org/wiki/Live_CD
[gpt]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[edk]: https://www.tianocore.org/
[cb]: https://www.coreboot.org/

## Partitioning

![A picture showing partitions on my USB drive.](/assets/usb-drive-partitions.png)

MBR is used as the partition table.

1. The first partition is the data partition in exFAT (MBR Partition Type `07h`). This allows the everyday use of the USB disk.
2. The EFI System Partition (ESP) in FAT32 (MBR Partition Type `EFh` ~~/ `1Bh`~~) is placed after the data partition.

This ESP looks quite weird: it's not the first partition, and it's on an MBR disk. In fact, this is completely legitimate[^2], maintaining compatibility. The problems behind actually come from Windows:

- On (slightly) older versions of Windows, only the first partition of a USB drive will be assigned a drive letter automatically, regardless of its type. Thus, placing the ESP at the first position results in the ESP being displayed rather than the data partition.
- Windows have never supported such layout. On recent versions of Windows 10, although the above issue is fixed (multiple partitions can be displayed normally), the ESP on MBR will still be assigned a drive letter and displayed, while it shouldn't be.

To mitigate the latter problem, the MBR partition type of ESP can be set to `1Bh` (Hidden FAT32). On my Dell laptop, it can still be recognised as a legitimate ESP (as it contains `\EFI\BOOT\BOOTX64.EFI`), but as Dell laptops can even boot from NTFS drives (such support is out of the specification), I think doing so might reduce usability. More tests are needed on this!

I formatted my data partition with exFAT (MBR Partition Type `07h`), which is a modern choice. The ESP can be FAT12, FAT16, or FAT32, but there is no reason for choosing the former two, unless _ancient DOS_ is required. Remember to add a file system label to the ESP if Arch ISO is needed (see below).

## Boot Loader

![A picture showing GNU GRUB 2 boot sequences.](/assets/grub2-boot-bios-uefi.png)

GNU GRUB 2 ("GRUB") is used as the boot loader. Two copies of GRUB are installed to provide support for both PC BIOS (`i386-pc`) and 64-bit x86 UEFI (`x86_64-efi`). When booting from PC BIOS, the GRUB bootstrap code in MBR boot sector is executed, then the stage 1.5 (`core.img`) is found and executed, then the `normal.mod` is found and loaded, then the configuration file is read. When booting from UEFI, the firmware loads `\EFI\BOOT\BOOTX64.EFI`, which _is_ GRUB EFI image, and then the normal flow continues. Both modes share one `/grub/grub.cfg`.

As GRUB relies on `grub.cfg` to set up itself, first we need some lines of boilerplate. The below is mine, but if you want to further customise it, [RTFM][grub-manual].

```conf
set gfxmode=auto
set gfxpayload=keep
set pager=1

insmod all_video
insmod gfxterm

loadfont unicode
load_video

terminal_input console
terminal_output gfxterm

search --fs-uuid --set=root A1B2-C3D4
```

... where `A1B2-C3D4` needs to be replaced with the real UUID of ESP. The `$root` variable tells GRUB that `/` is relative to that device.

[grub-manual]: https://www.gnu.org/software/grub/manual/grub/html_node/index.html

### Unrelated

In my previous design, [Systemd-Boot][sd-boot] was used as a simple boot loader for only the UEFI platform. Since GRUB supports multiple platforms, is more powerful, and can be managed more easily, this time I choose GRUB.

Since GRUB is big, it cannot accommodate itself solely in the MBR boot sector (which only has 446 byes). Thus, GRUB installed in PC BIOS mode puts its `core.img` on the unused sectors right after the 512-byte MBR. The first partition in MBR always starts at sector number 63 (track 1), leaving the former sectors, except the MBR sector, empty. It's also worth mentioning that GRUB boot in MBR mode is also capable of reading GPT, but GPT itself starts at sector 2, occupying the space where GRUB puts code. To overcome, in GPT there's a partition type called "BIOS Boot Partition" for accommodating `core.img`. One must have this partition (usually around 1MiB) set up before installing a PC BIOS GRUB on a GPT disk. I tried this before; it works great for Linux OSes, but Windows PE under PC BIOS doesn't know this, and will die immediately. We don't do this now.

## Operating Systems

The boot loader bootstraps the computer and shows a menu in order to bring up the Operating Systems (OSes).

Live systems are generally available in the ISO image format. As we need to make a multi-boot USB drive, we need to extract files from them. This can be done by:

- Double-clicking the ISO image in Explorer on Windows (10 or later), then copy.
- `mount -o ro,loop path/to/iso path/to/mount_point` on Linux, then copy.
- Using various 3rd-party software to extract.

Properly placing the required file is one of the task we need to pay attention to.

Below I've listed my choices, but you could use these methods to extract other Live systems too.

### Arch Linux Installation Medium

"Arch ISO" in short, this is the Live environment for installing Arch Linux, available on their [download page][archiso]. There are a few reasons why I choose this:

- Arch Linux is a rolling Linux distribution, and Arch ISO is updated automatically once a month. The software and Linux Kernel are always the latest.
- Arch ISO includes many handy system maintenance tools (of course).
- ~~Sometimes, people may want Arch Linux.~~

Arch ISO looks like this:

- `/arch/`: The directory containing all Arch ISO required files.
- `/EFI/`: The EFI required system directory containing boot loaders. [Systemd-Boot][sd-boot] is used in Arch ISO.
- `/loader/`: [Systemd-Boot][sd-boot] configuration files.
- `/syslinux/`: [Syslinux][syslinux] and its configuration file. This is used in PC BIOS.
- `/shellx64.efi`: A UEFI Shell.

As we've configured our boot loader, we only need the `arch/` directory. Extract it to the ESP of our USB drive, then add an entry to `grub.cfg` (replace `${PART_LABEL}` with the real disk label):

```conf
menuentry 'Arch Linux Installation Medium' {
  echo 'Loading Kernel...'
  linux /arch/boot/x86_64/vmlinuz-linux archisobasedir=arch archisolabel=${PART_LABEL}

  echo 'Loading Initrd...'
  initrd /arch/boot/intel-ucode.img /arch/boot/amd-ucode.img /arch/boot/x86_64/initramfs-linux.img

  echo 'Booting...'
  boot
}
```

The kernel command line arguments are copied from Arch ISO. If the base directory (`arch/`) name is changed, then `archisobasedir=arch` needs to be changed too. I don't know if there is `archisouuid=`; if there is, then there's no need to rely on the disk label.

[sd-boot]: https://www.freedesktop.org/wiki/Software/systemd/systemd-boot/
[syslinux]: https://wiki.syslinux.org/wiki/index.php?title=The_Syslinux_Project
[archiso]: https://archlinux.org/download/

### GParted Live

[GParted Live][gparted-live] is based on Debian, released along with GParted (GNOME Partition Editor). I find the GParted GUI straightforward, so having this in my swiss knife can be handy.

GParted Live looks like this:

- `/boot/`: GRUB.
- `/EFI/`: The EFI required system directory containing boot loaders.
- `/live/`: The directory containing all GParted Live required files.
- `/syslinux/`: Syslinux.
- `/utils/`: Some scripts and programs helping users to burn the ISO to a drive.
- Some licenses and versioning information.

Once again, we copy `live/` to our ESP. However, bare in mind that the name of directory is quite generic; it's actually the default name used by [`live-boot`][livesys]. Many Debian-based OSes use this name too (e.g. Tails), so to avoid collision, we need to change it to `gparted-live`, or any other name. Then, we use the `live-media-path=/gparted-live` kernel parameter to tell `live-boot` to find the root file system under `/gparted-live`.

```conf
menuentry 'GParted Live' {
  echo 'Loading Kernel...'
  linux /gparted-live/vmlinuz boot=live union=overlay username=user config components quiet noswap ip= net.ifnames=0 nosplash live-media-path=/gparted-live

  echo 'Loading Initrd...'
  initrd /gparted-live/initrd.img

  echo 'Booting...'
  boot
}
```

For other Debian-based Live ISOs, e.g. Tails, do the same thing.

[gparted-live]: https://gparted.org/livecd.php
[livesys]: https://live-systems.org/

### Windows Pre-installation Environment (PE)

A Windows PE can be used to maintain Windows. I use a Windows PE to install Windows 10:

- `install.wim` in today's Windows 10 ISO has exceed 4GiB, which is the individual file size limit for FAT32, which is required by the UEFI specification. It's possible to split the image, but this is tedious.
- Installing Windows 10 via Windows PE manually is not difficult, and _advanced_ operations can be done.
  - ... mainly to avoid `setup.exe` bugs.

I get a vanilla Windows 10 PE (2004). A detailed guide on how to get it can be found [here][winpe]. After `MakeWinPEMedia /ISO`, we extract required files to our ESP:

- `/bootmgr` and `/bootmgr.efi` could be the first file Windows tries to load.
- `/Boot/` contains PC BIOS BCD (Windows boot loader) files.
- `/EFI/Microsoft/` contains UEFI BCD files.
- `/EFI/Boot/bootx64.efi` contains the first-stage Windows UEFI loader.
  - this is copied to `/EFI/Microsoft/bootx64.efi` to avoid collision
- `/sources/`: contains `boot.wim`, which contains Windows PE.

Then `grub.cfg`. Due to the different loading method in different boot modes, they're split into two separate menu entries:

```cfg
menuentry 'Windows 10 PE (BIOS Only)' {
  echo 'Loading Bootmgr...'
  ntldr /bootmgr

  echo 'Booting...'
  boot
}

menuentry 'Windows 10 PE (UEFI Only)' {
  echo 'Loading Bootmgr...'
  chainloader /EFI/Microsoft/bootx64.efi

  echo 'Booting...'
  boot
}
```

3rd-party customised Windows PEs should follow the same method above.

[winpe]: https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro

### MemTest86+, MemTest86

[MemTest86+][memtest86p] (PC BIOS only) and [MemTest86][memtest86] (UEFI only) are not OSes, but memory testing software that is frequently used to diagnose stability problems. They both comes from the same code base, but are now maintained by different groups of people. Having them in the swiss knife is quite useful.

MemTest86+ is made to be directly bootable under PC BIOS; it does not support UEFI. The "pre-compiled bootable binary" option can be downloaded directly from the home page. Uncompress the image, place it into `/memtest86+/` in our ESP, and add an entry to `grub.cfg`:

```cfg
menuentry 'MemTest86+ (BIOS Only)' {
  echo 'Loading MemTest86+...'
  chainloader /memtest86+/memtest86+-5.31b.bin

  echo 'Booting...'
  boot
}
```

Replace `memtest86+-5.31b.bin` with the actual file name.

PassMark (who maintains MemTest86 today) makes the download easier for users to _burn_, but we only need the files. To only fetch the files, Linux is required. The 500MiB `memtest86-usb.img` file contains a 250MiB NTFS partition and a 250MiB FAT32 partition (ESP) with identical files. We put the required files under `/memtest86/`.

```shell
losetup -fP path/to/memtest86-usb.img # Assume loop0 is allocated
mount /dev/loop0p1 /mnt
cp -r /mnt/EFI/BOOT/* path/to/our/ESP/memtest86/
```

Then `grub.cfg`:

```cfg
menuentry 'MemTest86 (UEFI Only)' {
  echo 'Loading MemTest86...'
  chainloader /memtest86/bootx64.efi

  echo 'Booting...'
  boot
}
```

[memtest86p]: https://www.memtest.org/
[memtest86]: https://www.memtest86.com/

### Other OSes

Besides the forehead mentioned OSes, I also have these in my USB drive:

- [Tails][tails]. This is the Edward Snoden endorsed OS featuring forced Tor connection across the system to reach a higher level of anonymity when browsing on the dark net. It ships with many applications for paranoids' daily use. I don't frequently use it; sometimes I do my little researches on it.
- [Slax Linux][slax]. This is a generic desktop Linux system designed to be portable. It features AUFS (although OverlayFS do better work nowadays) that can split applications into individual modules that can be conveniently loaded. By default, it comes with a Chromium browser, which should be enough for temporary browsing.
  - Because of the portable design, placing it on our ESP is not a big deal.

[tails]: https://live-systems.org/
[slax]: https://www.slax.org/

## Final Words

Handcrafting a swiss knife enhances one's understanding surrounding it, but I write this blog mainly for showing off the design, since I've really invested some time on it. :)

## References

[^1]: Rod Smith, [Hybrid MBRs: The Good, the Bad, and the So Ugly You'll Tear Your Eyes Out](https://www.rodsbooks.com/gdisk/hybrid.html).
[^2]: "If an MBR partition has an OSType field of 0xEF (i.e., UEFI System Partition), then the firmware must add the UEFI System Partition GUID to the handle for the MBR partition using InstallProtocolInterface()." [UEFI Specification, Version 2.8 Errata B (big PDF)](https://uefi.org/sites/default/files/resources/UEFI%20Spec%202.8B%20May%202020.pdf).
