---
layout: post
title:  "A Fundamental Guide to Linux"
date:   2019-10-17
---

Hi! In the next few chapters we will learn some **very basic** usage of Linux operating systems. Nowadays Linux operating system(s) are widely used among cloud servers and high performance computing (including the GPU server we may use), but Linux is quite different from Microsoft Windows, which we use every day, so the concepts may seem difficult for us. But don't worry, should you encounter any problem about Linux that is not covered in this document, feel free to ask me!

## What is Linux

> "Linux is not another DOS."
>
> -- Dr. Jefferson Fong

In 1991, Linus Torvalds created the Linux operating system in University of Helsinki in Finland. What makes Linux so successful is its openness: its source code is open for access and free (both as in "freedom" and as in "free beer"), and its code is constantly changing thanks to millions of volunteers around the world (which means we can change Linux too if we have such ability).

Linux _per se_ is only an operating system **kernel**, not a usable operating system. (Recall the COMP3033 Operating System course!) Thus, people around the world write peripheral software (a.k.a. user-space programs) and pack them together. Operating systems with Linux kernel that are usable out-of-box are called **Linux distributions (a.k.a. distros)**. Famous Linux distributions include _Ubuntu_, _Fedora_, _CentOS_, _Arch Linux_, and (thousands) more.

The GPU server provided for our use is installed with _Ubuntu 18.04_, which is a stable and modern one.

## Log into a Remote Linux Operating System

To log into a remote Linux operating system, we use the **Secured SHell (`ssh`)**. We need to first have an SSH client installed on our local operating system. Here are some options:

- On Microsoft Windows:
  - [PuTTY][putty] is recommended.
  - A recent version of Windows 10 (1809 and newer) should already have a `ssh` installed. To use it, follow the following steps:
    - Press `<WINDOWS> + X`
    - In the popped-up context menu, click **"Windows PowerShell"** (or **Command Prompt**).
    - Start using `ssh` as is described below.
  - SSH is also built into Git Bash, [MSYS2][msys2], and [Cygwin][cygwin].
- On macOS: SSH is pre-installed.
- On Linux: SSH is usually pre-installed. If not, ask Google.
  - In Microsoft Store, Linux distributions built on [Windows Subsystem of Linux (WSL)][wsl2] have the same usage as normal Linux operating systems.
- On Android: [Termux][termux], [ConnectBot][connectbot].
- On iOS and iPadOS: [Termius][termius].

[putty]:      https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
[msys2]:      https://www.msys2.org/
[cygwin]:     https://www.cygwin.com/
[wsl2]:       https://docs.microsoft.com/en-us/windows/wsl/wsl2-about
[termux]:     https://play.google.com/store/apps/details?id=com.termux
[connectbot]: https://play.google.com/store/apps/details?id=org.connectbot
[termius]:    https://apps.apple.com/us/app/termius-ssh-client/id549039908

In this document, only the standard `ssh` command line program is covered. Information about the GPU server is described in the _"Introduction to GPU server"_ (J. Wu, J. Yu, L. Zheng, et al.) document.

Next, connect to the server:

```shell
ssh <name>@nndl.uiccst.com
```

... where:

- `<name>` is our account name (on our GPU server this is our student ID).

The following is an example of first login:

```
$ ssh 1630003072@nndl.uiccst.com
The authenticity of host 'nndl.uiccst.com (172.31.19.19)' can't be established.
ECDSA key fingerprint is SHA256:oSlAV0yVYODHi6EOLQujjOSVtBg+aYbgd4eRmmj+nMY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'nndl.uiccst.com,172.31.19.19' (ECDSA) to the list of known hosts.
1630003072@nndl.uiccst.com's password:
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-65-generic x86_64)

...

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

1630003072@cst-ml-server001:~$
```

On first connection, the SSH client asks if we trust the remote machine. We just trust it. Then we should see the command line prompt (the `$` means we are currently logged in as a normal user; `#` means super user _root_).

## Look Around

Look at the texts before the cursor. They mean:

```
<username>@<hostname>:<currdir>$
```

... where:

- `<username>` is the user name.
- `<hostname>` is the name of the connected machine.
- `<currdir>` is the current location. There are some special values:
  - `/` is the root directory (see below).
  - `~` is our home directory; we should put all personal files here.

Files and directories in Linux operating systems (and all those UNIX-like systems) are organized in a **tree** form. `/` is called the "root directory", under which directories are **nodes** with children, and files are **leaves**. This is different from Microsoft Windows: there is no `C:` or `D:` thing.

To list files and directories in the current location, type:

```shell
ls # a brief list
ll # a detailed list (alias of `ls -lh`)
l  # a more detailed list with hidden files listed (alias of `ls -lah`)
```

To go into a directory, type `cd <path>`, where `<path>` is the path to the directory. Examples:

```shell
# Enter the "training" directory in the "data" directory
# under the current location
cd data/training

# Enter the system configuration directory, which is located
# under the root directory
cd /etc

# Go back to the home directory (yes, just type `cd`)
cd
```

To create a directory, type `mkdir <name>`, where `<name>` is the name of directory. The directory will be created under the current location; to create directory in other directory, type `mkdir path/to/dir/<name>`.

To copy a file or a directory, type:

```
cp [-r] <source> <target>
```

... where `<source>` is the file we want to copy, and `<target>` is the location we want the file goes. The **optional** `-r` tells `cp` to copy files _**r**ecursively_; otherwise `cp` refuses to copy a directory.

```shell
# Copy train.txt from the current directory to the data directory
# The trailing slash is not required
cp train.txt data/

# Copy the whole directory named "data" to .keras/mnist
# The trailing slash is not required
cp -r data/ .keras/mnist
```

To move a file (instead of copy), use `mv`. Note, that to rename a file or directory, `mv` is also used:

```shell
# Move train.txt to directory data (the original file will disappear)
# The trailing slash is not required
mv train.txt data/

# Rename directory data to dataset
mv data dataset
```

To delete a file, type `rm <file>`, where `<file>` is the name of the file we want to delete.

To delete a directory, type `rm -r <dirname>`, where `<dirname>` is the name of the directory we want to delete. The `-r` argument means "delete the directory _**r**ecursively_"; otherwise `rm` refuses to delete a directory.

## Text Processing

One of the powerful and fascinating things about Linux operating systems is that the system utilities, when combined, are extremely powerful. Let's see how to use some of the tools to do cool things.

Suppose there is a big text file named "train.log", and we want to find text lines containing the string "accuracy". Type:

```shell
cat train.log | grep "accuracy"
```

Let's break this out:

- `cat` means "con**cat**enate" ~~(not cats!)~~, but if only one file is provided, it just prints it. So `cat train.log` prints the file content of `train.log`.
- `|` is a **pipe**. Recall what we learned in COMP3033 Operating System course: _pipe_ copies the _standard output_ from the former command, and pastes it to the _standard input_ of the latter command. So the _pipe_ copies the file content of `train.log` and passes it to `grep`.
- `grep "accuracy"` searches for the string `accuracy` from its _standard input_. If it finds one, it prints the whole line containing the string.

To edit files on the server, use `nano`, which is simple to use. Just append the path to the file we want to edit. To exit `nano`, press `<CTRL> + X`. `vim` can also be used, which is more advanced, but hard to use (it inherits the operation mode back from 1970s). In case we accidently get stuck in the `vim` editor, remember how to exit the editor: first press `<ESC>`, then press `Z`, finally press `Q`.

## File Exchange

Sometimes we may want to download files on the server or upload files to the server. SSH has file transfer support, so we just use it.

**Secured CoPy (`scp`)** is used to transfer files from and to SSH-enabled servers. Basic usage of this command:

```
scp <from> <to>
```

... where:

- `<from>` is the path to the file to be copied:
  - if it's on our local computer, just type in the path pointing to the file
  - if it's on the server, type as in this format: `<username>@<hostname>:path/to/file`
    - mostly the same as `ssh`
    - if the path is not absolute (e.g. `data/train.txt`), `scp` thinks it in our home directory
- `<to>` is the path to the copied location
  - basically follows the rules above

An example:

```
$ scp train.txt 1630003072@nndl.uiccst.com:
1630003072@nndl.uiccst.com's password:
train.txt                                          100%    0     0.0KB/s   00:00
```

This copies `train.txt` from the local, current directory to `1630003072`'s home directory in the server (note that there is nothing after the colon).

There is a more convenient way: use [FileZilla][filezilla], which should be familiar to us. When typing the remote hostname, remember to prefix `sftp://`, which means the file transfer protocol running on SSH. The port should be `22`.

[filezilla]: https://filezilla-project.org/

## System Information

In case we wonder the usage of the GPU server, type `htop` and we can see a task manager. The task manager tells us the CPU usage, memory usage, and more.

To see the GPU usage, type `nvidia-smi`. This tool should be straightforward.
