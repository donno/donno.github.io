---
layout: post
title:  "CrosVM on Windows"
date:   2025-05-18 16:00:00 +1030
---

This weekend's little project was looked at playing with the [crosvm project][0] on
Microsoft Windows. The crosvm project is a type-2 virtual machine monitor,
also known as a hosted hypervisor meaning that it runs on top of a conventional
operating system.

Its full name is the ChromeOS Virtual Machine Monitor, and it was developed for
running Linux as guests on ChromeOS devices. It supports Windows by using the
[WHPX (Windows Hypervisor Platform)][2].

Building
--------
The project is fairly straight forward to [build on Windows][3], if you already have
a suitable version of Visual Studio installed for Rust development.

While the process to building is well documented on [Book of crosvm][3], I'll repeat
the steps I ran.

### Fetch the source code
```sh
git clone https://chromium.googlesource.com/crosvm/crosvm
cd crosvm
git submodule update --init
git config submodule.recurse true
git config push.recurseSubmodules no
```
The config settings help ensure the submodules are kept in sync with the main repository.

### Build the source
```
Set-ExecutionPolicy Unrestricted -Scope CurrentUser
./tools/install-deps.ps1
cargo build --release --features all-msvc64,whpx
```

Running
-------
To run the program, you need a Linux kernel and an initial RAM disk. Their instructions
suggest using `virt-builder` to create the image, however that doesn't function well from
either a container or WSL2 (Windows Subsystem for Linux 2). as both lacks the kernel of the
host.

The approach taken was to grab both components from  `alpine-virt-3.21.3-x86_64.iso` which
provides the `initram-fs-virt` and `vmlinuz-virt` files. This looks promising as the kernel
boots however there is no root disk. It therefore enters recovery mode.

In this case I'm running it through cargo, but I could just as easily go directly to the executable.
```
cargo run --release --features all-msvc64,whpx -- run --cpus 2 --mem 512 --initrd initramfs-virt vmlinuz-virt
```

### Root filesystem
There were three approach that I took to try to source a root filesystem for it.

* Create one myself from a alpine-minirootfs tar file.
* Use the generic cloud image (`generic_alpine-3.21.2-x86_64-bios-tiny-r0.qcow2`)
* Use a script for creating a more inital RAM disk.

### Build own disk image
```sh
dd if=/dev/zero of=alpine.raw bs=1M count=200
mkfs.ext4 alpine.raw
losetup /dev/loop0 alpine.raw
mount -o loop alpine.raw /mnt/alpine
cd /mnt/alpine
tar xf ~/alpine-minirootfs-3.21.3-x86_64.tar.gz
umount /mnt/alpine
```
This fails and it doesn't look great when you use `fdisk -l` in the recovery mode.

### Generic cloud image
`--block generic_alpine-3.21.2-x86_64-bios-tiny-r0.1.qcow2`

This looked quite promising:
```
[    1.024704] Run /init as init process
[    1.032226] Alpine Init 3.11.1-r0
Alpine Init 3.11.1-r0
[    1.033676] Loading boot drivers...
 * Loading boot drivers: [    1.041538] loop: module loaded
[    1.070969] ACPI: bus type drm_connector registered
[    1.086951] Loading boot drivers: ok.
ok.
[    1.089048] Mounting boot media...
 * Mounting boot media: [    1.120226] virtio_blk virtio0: 2/0/0 default/read/poll queues
[    1.123323] virtio_blk virtio0: [vda] 245760 512-byte logical blocks (126 MB/120 MiB)
[    1.271493] EXT4-fs (vda): mounted filesystem 1d7db88a-6f75-4f93-ab3e-df6b8e3031de ro with ordered data mode. Quota mode: none.
[    1.278785] EXT4-fs (vda): unmounting filesystem 1d7db88a-6f75-4f93-ab3e-df6b8e3031de.
[    6.285662] Mounting boot media: failed.
failed.
initramfs emergency recovery shell launched. Type 'exit' to continue boot
sh: can't access tty; job control turned off
```

The output of `fdisk -l` is odd.
```
~ # fdisk -l
Disk /dev/vda: 120 MB, 125829120 bytes, 245760 sectors
120 cylinders, 64 heads, 32 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device  Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/vda1 09 187,180,14  784,0,13    3224498923 3657370039  432871117  206G  7 HPFS/NTFS
Partition 1 has different physical/logical start (non-Linux?):
     phys=(187,180,14) logical=(1574462,23,12)
Partition 1 has different physical/logical end:
     phys=(784,0,13) logical=(1785825,13,24)
/dev/vda2 f4 906,235,61  262,116,59  3272020941  930513678 1953460034  931G 16 Hidden FAT16
Partition 2 has different physical/logical start (non-Linux?):
     phys=(906,235,61) logical=(1597666,30,14)
Partition 2 has different physical/logical end:
     phys=(262,116,59) logical=(454352,24,15)
/dev/vda3 20 370,101,50  10,114,13            0          0          0     0 6f Unknown
Partition 3 has different physical/logical start (non-Linux?):
     phys=(370,101,50) logical=(0,0,1)
Partition 3 has different physical/logical end:
     phys=(10,114,13) logical=(2097151,63,32)
/dev/vda4    0,0,0       0,0,0         50200576  974536369  924335794  440G  0 Empty
Partition 4 has different physical/logical start (non-Linux?):
     phys=(0,0,0) logical=(24512,0,1)
Partition 4 has different physical/logical end:
     phys=(0,0,0) logical=(475847,53,18)
```

### Build inital RAM disk
Here we use the [`firecracker-initrd`][4] by `marcov`, which is for creating an
Alpine-based initrd for Firecracker (which is Amazon's project similar to crosvm).

The new image produced by this script, replaces the image provided by the `--initrd` argument.
This has a working system so it doesn't just bail out the recovery mode.

```
[    0.934314] Run /init as init process

   OpenRC 0.55.1 is starting up Linux 6.12.8-0-virt (x86_64)

 * Mounting /proc ... [ ok ]
 * Mounting /run ... [ ok ]

... [omitted several other lines] ...

Welcome to Alpine Linux 3.21
Kernel 6.12.8-0-virt on an x86_64 (ttyS0)

localhost login:
```

The above project set-ups a `sshd` so you can `ssh` into the machine, however
without networking that is difficult. This provided a road black as
the networking for the virtual machine wasn't set-up and the documentation
for Windows does not cover the networking. Neither was it possible to find any
reports or blogs of others who had done this.

This is where I thought I would end this article but I ended up rechecking the
script and noticed that it does set the password for root, so the login is
`root` with the password `root`.

As seen below, it is possible to log in to the system and once in you can then
power it off (shut it down).

```
localhost login: root

Password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

login[1006]: root login on 'ttyS0'
localhost:~# ls /
bin    etc    init   media  opt    root   sbin   sys    usr
dev    home   lib    mnt    proc   run    srv    tmp    var
localhost:~# poweroff
localhost:~#
 * Unmounting loop devices
 * Unmounting filesystems
 * Stopping fcnet ... * start-stop-daemon: no matching processes found
 [ ok ]

The system is going down NOW!

Sent SIGTERM to all processes

Sent SIGKILL to all processes

Requesting system poweroff
[  153.860672] ACPI: PM: Preparing to enter system sleep state S5
[  153.861722] reboot: Power down
```

Summary
-------
Great little project as the resulting binaries are about 12MB.
The 39MB for the Linux kernel and RAM disk brings it up to just over 50MB
for a working system on Windows. That is assuming that you already have
Windows Hypervisor Platform installed and set-up.

My machine was already using both Hyper-V and WSL2 so I don't know exactly
what it would take to set-up on a fresh Windows 10 (or 11) machine.

Future
------
* Working networking would be great, to be able to install packages from Internet (or network)
  and to be able to interact with the Internet.
* Revisit the block device to have a place to have persistent storage.
* Rework the initrd script to allow:
  * Choosing what packages to configure
  * Adding extra users - potentially importing the SSH keys from Launchpad or GitHub via [`ssh-import-id`][5].
* Figure out if there is a way to get the release version of `r8Brain.dll` rather than debug as it
  requires the debug version of the Microsoft Visual C++ runtimes. The library is for audio support
  on Windows.

If you want to learn more than about crosvm then check out Daniel Prilik's post on [the subject][6].

[0]: https://crosvm.dev/book/
[1]: https://chromium.googlesource.com/crosvm/crosvm/
[2]: https://learn.microsoft.com/en-us/virtualization/api/hypervisor-platform/hypervisor-platform
[3]: https://crosvm.dev/book/building_crosvm/windows.html
[4]: https://github.com/marcov/firecracker-initrd
[5]: https://launchpad.net/ssh-import-id
[6]: https://prilik.com/blog/post/crosvm-paravirt/
