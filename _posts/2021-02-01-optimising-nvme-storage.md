---
layout: post
title:  "Optimising NVMe Storages"
date:   2021-02-01
---

**TL;DR: Change the LBA format to an optimal one (usually with a block size of 4KiB).**

[Jump to the most relavant section.](#lba-formats)

Modern storage devices utilising the [NVMe][nvme] interface enables more flexible configurations and much faster operations on Non-Volatile Memories (e.g. Solid-State Drives (SSDs)). NVMe was originally developed for enterprise-class storages, but upon completion it's became a specification suitable for both consumer-class and enterprise-class devices. As a result, NVMe controllers can implement tons of advanced features within the NVMe specification, but most of them will only present on enterprise-class NVMe disks. Often, only a very limited set of features could be tweaked on consumer-class NVMe SSDs, but high-end ones usually have more tweakable parameters.

[nvme]: https://en.wikipedia.org/wiki/NVM_Express

## Some Backgrounds

There are a number of types of storage devices in the market, confusing consumers. One thing we need to bear in mind is that these technologies are often stacked (combined) together to satisfy different needs. I tried to make some simple clarifications and comparisons among common terminologies below. Obsolete standards (in 2021) are omitted.

### Physical Connectors

- **SATA** (Serial AT Attachment): (Still) the most common interface for consumer storage devices (in 2021), equipped mostly on 2.5" or 3.5" HDDs (Hard Disk Drives; those spin internally) and budget SSDs.
  - **mSATA** (Mini-SATA): This is nearly obsolete, but is still around the market. Before NVMe and M.2 (see below) became dominant, mSATA was the popular one. This is a standard for connecting SATA drives to a mPCIe connector, so it looks like a standard mPCIe expansion card, but is not electronically compatible with mPCIe. Not all computers (after 2011) support mSATA.
- **M.2** a.k.a. **NGFF** (Next Generation Form Factor): a specification for internally mounted expansion cards, replacing mSATA.
  - Two defined types of connector[^1] are commonly used: **B Key** and **M Key**. Their combination, **B + M Key**, also exists. These are used by the motherboard manufacturers to denote their supported modes of connection; incompatible cards with different keying simply can't be inserted.
  - Different form factors are also defined. For example, _2242_ means a _22mm wide, 42mm long_ M.2 expansion card.
  - On top of the connector, many protocols can run, including PCIe, SATA, and USB. For some protocols, different modes can be distinguished from the connector keying mentioned above, e.g. B Key runs PCIe x2, and M Key runs PCIe x4.

### Data Link Interfaces (Protocols)

- **SATA** (again): Despite the physical layer protocol, SATA also defines its own data link layer protocol and trasport layer protocol. It can run on not only SATA connectors (family), but also PCIe (called mSATA), M.2, U.2, etc.
- **NVMe**: a specification for SSDs to run on top of PCIe.
  - Form factors are not defined, meaning that NVMe can run on any connector as long as PCIe is used. For situations where PCIe is not present, a separate specification called NVMe-oF (NVMe over Fibre; not restricted to _Optical Fibres_) is defined for transmitting NVMe commands over computer networks.
    - The NVMe 2.0 specification (not released yet) will combine NVMe (base) and NVMe-oF.

The following are unrealted terminologies, but are worth mentioning. These are specifications for communications between the operating systems and the storage systems, regardless of what the storage system looks like.

- **AHCI** (Advanced Host Controller Interface): A _de facto_ specification made by Intel for exposing SATA capabilities in an implementation-independent manner through the SATA host controller.
- **"RAID"** (as is written on most firmware settings): _Is_ **Intel Rapid Storage Technology** (RST), firmware-supported [RAID][raid]. Like traditional RAID systems, two or more storage devices are required to utilise this.

[raid]: https://en.wikipedia.org/wiki/RAID

### NVMe Namespaces

On Linux, NVMe devices are displayed as `nvmeXnYpZ`, much more complicated than the AHCI ones (`sdXY`). This is because of the PCIe-dependent architecture of NVMe: controllers are on the PCIe bus. Thus, a computer system can have multiple NVMe controllers simultaneously connected. The number of controller assigned by Linux is denoted by the `X`.

NVMe namespace is an exciting feature allowing multiple storage backends to be exposed by one controller. With namespaces, NVMe controllers may implement interesting features, such as thin provisioning and data sharing across controllers. The number of namespace (NSID reported by the controller) is denoted by the `Y`. Basically, a namespace is a disk device.

The number of partition is denoted by the `Z`. Thus, `nvme0n1p2` is _the second partition in the **first** (starts from 1) namespace exposed by the **first** (starts from 0) NVMe controller_.

## LBA Formats

Nowadays, storage devices generally use 4KiB (4096 bytes) as the read / write unit (_physical block size_), which reduces overhead brought by checksums and improves speed. However, they usually provide an emulation layer too, which announces a 512-byte _logical block size_, for compatibility reasons. The 4K alignment problem exists because of such setting: if a partition is not 4K-aligned, more physical blocks will be operated, reducing the life span of flash memories. NVMe storage devices are no exception, but unlike SATA or USB storage devices, we, as users, can adjust it.

Install `nvme-cli` on your Linux distribution. This gives you the `nvme` command. Suppose we have only one NVMe controller and one namespace behind the controller (this is also the unconfigurable setting for most consumer NVMe disks). The following command gives the capabilities of the NVMe namespace (disk):

```shell
nvme id-ns --namespace-id=1 --human-readable /dev/nvme0

# Alternative equivalent in shorthands
# nvme id-ns -H /dev/nvme0n1
```

We only focus on the last few lines, since NVMe is a so sophiscated specification, as you may see. The last few lines starting with "LBA Format" outputs the supported Logical Block Address Format (LBAF) of the namespace. If there is only one entry, then there's nothing you can do -- it's already "optimal". For my NVMe drive, I get the following output:

```
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x3 Degraded
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better (in use)
```

The two entries indicate that my NVMe namespace (disk) offers two options: a compatibility LBAF with a block size of 512B, and a performance LBAF with a block size of 4KiB. The smaller the Relative Performance is, the better. The format number ("LBA Format _X_") is used to identify it in the format command (see below).

To switch the LBAF from "degraded" (format 0) to "better" (format 1), a format operation sent by `nvme` is required. Be sure to backup anything on the disk, then invoke:

```shell
nvme format --namespace-id=1 --lbaf=1 /dev/nvme0

# In short
# nvme format -l 1 /dev/nvme0n1
```

`nvme` will make you think twice by pausing 10 seconds. Do think twice! The operation finishes in seconds, after which your NVMe disk will be cleared, and should perform better (if you just switched to the "better" LBAF). Today nearly all software and modern operating systems support 4K logical block size, so unless you run _ancient_ software, you don't need to think about the compatibility.

## See Also

- [A Quick Tour of NVM Express (NVMe)](https://metebalci.com/blog/a-quick-tour-of-nvm-express-nvme/)

## Notes

[^1]: There are twelve (12) keys defined in the M.2 specification, ranging from **A Key** to **M Key** (_I_ is excluded). The specifications on PCISIG have restricted access, but a version 1.0 one can be accessed [here][pcie-m2-spec-1.0]. Figure 64 on Page 79 illustrates these keys.

[pcie-m2-spec-1.0]: http://read.pudn.com/downloads794/doc/project/3133918/PCIe_M.2_Electromechanical_Spec_Rev1.0_Final_11012013_RS_Clean.pdf
