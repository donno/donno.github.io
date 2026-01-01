
After looking at QEMU's support for the P9 file system protocol, its support
for [Network Block Device (NBD)][0], caught my eye.

## Experiment

The QEMU project includes a Network Block Device server called [`qemu-nbd`][1],
which can export a QEMU disk image using the [NBD protocol][2]. The
[documentation][1] for it has several examples including how to set-up the
use of a certificate for authentication but I'm just intrested in the
basics.

### Server

Create a simple 1 GiB image in the QCOW2 format (QEMU's native format).
```sh
qemu-img create -f qcow2 nbd_demo.qcow2 1G
```

Host the image:
```sh
qemu-nbd -f qcow2 nbd_demo.qcow2 --name example
```

* Exports the given image with the export name (`example`).
* After the first successful client disconnects the server will exit.
* There is no TLS encryption

On Alpine both `qemu-img` and `qemu-nbd` are included in the `qemu-img`
package.

### Client

The obvious client for the above server is QEMU itself. QEMU supports
QEMU supports Network Block Devices using TCP protocol and Unix Domain Sockets.

A device can be provided to qemu-system executable using TCP as follows:
```--drive file=nbd://192.0.2.1:30000```
If the export was named then the name of the export forms hte path, i.e.
`/qemu` based on the example above.

My interest in NBD was for the network so the Unix Domain Sockets weren't of
interested and being tied down to QEMU wasn't what I wanted wanted.

The Linux Kernel has a [module][3] for using NBD over the network.

```sh
apk add nbd-client
nbd-client 192.168.20.29 10809 --name example
```

Now it should have connected it as `/dev/nbd0` and should say the size.

```
Negotiation: ..size = 1024MB
Connected /dev/nbd0
```

Check with `fdisk -l /dev/nbd0`, it should report the size is 1GiB (or
at least match the size of the image you created).

Example output:
```
Disk /dev/nbd0: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

The benefit of NBD over Network File System (NFS) is the former lets you
format the device to what ever format. In my case I'm going to `vfat` as
while I Would have preferred `minix` to continue my previous experiments,
Windows Subsystem for Linux kernel doesn't include the `minix` driver.


Now format the device
```sh
mkfs.vfat /dev/nbd0
```

Mount the file system and populate it with some files:
```
mount /dev/nbd0 /media/network
mkdir /media/network/ast /media/network/erik /media/network/programs
echo "Hello Andrew"  /media/network/ast/welcome
```

When you are done, unmount and disconnect:
```sh
umount /dev/nbd0
nbd-client -d /dev/nbd0
```

You should be able to restart `qemu-nbd` and connect again and mount it and
see the changes you made last time.

### Troubleshooting
When running `nbd-client`,on Debian, the following error was shown:
```
Error: Couldn't resolve the nbd netlink family, make sure the nbd module is loaded and your nbd driver supports the netlink interface.
```

The fix was to run `modprobe nbd`.

## Alternative Server
I came across [`nbdserv`][4] which is a simple sever written in Rust using the
authors NBD [library/create][5]. This was ideal as the licence
is MIT/Apache 2.0.

The other two Rust-based implementation I came across were:
* The production quality implementation with the kitchen sink (it provides
  NFS and 9P access in addition to NBD) is provided by the [ZeroFS][6] project
  which is AGPL3 with option for commercial licencing.
* [`tokio-nbd`][7] which handles performing asynchronous I/O and has a
  reasonable good looking interface (trait) for providing your own backend. It
  is GPLv2 so that put me off this one as well as I'm not willing the close
  that particular door either.

For `C++` there is [`lwNBD`][9] (2-Claused BSD licence) aimed at low-end
hardware and [`nbdkit`][10], which supports plugins and filters in C, C++, Lua,
Python and a range of other languages and is licenced under 3-clause BSD
licence.

### Patch
Submitted a [patch][8] for `nbdserv` to update the `nbd` crate as the API
changed in order to support multiple exports. This marks the first pull request
that I submitted for a project in Rust.

## Alternative Client

QEMU system using direct Lunux boot with drive from NBD. This I couldn't get working
qemu-system-x86_64 -kernel alpine.kernel -initrd alpine.initrd --drive file=nbd://92.168.20.29:10809
```
```
Another tool encountered was [`guestfish`][14] from [`libguestfs`][15].
The latter project is a is a set of tools for accessing and modifying virtual
machine (VM) disk images. The former lets you exxmain and modifier virtual
machine file systems and it supports connecting to a device over NBD. I didn't
try this tool.

## Requests
In order to define your own type for backing the block device, you implement
the Read, Write and Seek traits from Rust. To get started, I simply defined
my own struct and implemented it where they call onto the file that way
everything still works.

This is the nice consequence of being able to add logging into the function
calls to get insight into how its working without diving deeper into the
protocol or NBD client.

Straight after connecting these are the seeks and reads that were done:
`nbd-client 192.168.20.29 8998 --name rustnbd`

* Seek from start: 0
* Read 4096 bytes
* Seek from start: 52363264
* Read 4096 bytes
* Seek from start: 52420608
* Read 4096 bytes
* Seek from start: 4096
* Read 4096 bytes
* Seek from start: 52424704
* Read 4096 bytes
* Seek from start: 52293632
* .. Several other seeks and read
* Seek from start: 139264
* Read 65536 bytes
* Read 57344 bytes


After mounting the file system which was ext3, it started reading the smaller
parts, i.e. 1024 bytes from here and there.

```sh
$ blockdev --getbsz /dev/nbd2
1024
```

For Alpine, install `e2fsprogs-extra` for `dumpe2fs` to dump the file system
information (`dumpe2fs /dev/nbd2`).
```
Block size:               1024
Fragment size:            1024
```

## Validation

Several tools can be chased down.

### lsblk

```
$ lsblk --raw --noheadings /dev/nbd4
nbd4 43:64 0 3.9M 0 disk
```

### fdisk

```
$ fdisk -l /dev/nbd4
Disk /dev/nbd4: 3.91 MiB, 4096000 bytes, 8000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4f0d50ee

Device      Boot Start   End Sectors  Size Id Type
/dev/nbd4p1          1  7999    7999  3.9M 83 Linux
```

### parted

```
$ parted -s /dev/nbd4 print
Model: Unknown (unknown)
Disk /dev/nbd4: 4096kB
Sector size (logical/physical): 512B/512B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  4096kB  4096kB  ext3
```

### fsck

This is useful to run after a `mkfs` command such as `mkfs.ext4`.
```sh
fsck /dev/nbd4
```

The output of it failing when there was a problem in my `read` function where
it wasn't honouring the current position after a seek.

```
fsck from util-linux 2.38.1
e2fsck 1.47.0 (5-Feb-2023)
Superblock has an invalid journal (inode 8).
```

Whereas here is the working case
```
fsck from util-linux 2.38.1
e2fsck 1.47.0 (5-Feb-2023)
/dev/nbd4: clean, 11/1008 files, 73/1000 blocks
```

### dd

Copy a file (i.e. disk image) to the block device.

```sh
dd if=example.raw of=/dev/nbd4 bs=8k conv=notrunc,sync,fsync oflag=direct status=progress
```

### mkfs.ext4
This creates a file with a block size of 4096 bytes and 1000 blocks formatted
as ext4.

```sh
mkfs.ext4 -b 4096 example.ext4 1000
```

Combining this with `dd` above it can then be applied to the image, or
mounted first to have extra files added.
```sh
dd if=example.ext4 of=/dev/nbd4 bs=8k conv=notrunc,sync,fsync oflag=direct status=progress
```

The case where this is handy is if you suspect you have a problem with
writing and seeking. This is because this command ensures it will write 8k at
a time to the block device where using `mkfs.ext4` directly on the block device
will cause it to do a bunch of seeking and writing.

### dumpe2fs

```sh
dumpe2fs /dev/nbd4
```

Output
```
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          18d5067c-c951-43a3-a810-80617c8167e6
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      ext_attr resize_inode dir_index filetype sparse_super large_file
Filesystem flags:         unsigned_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              1000
Block count:              4000
Reserved block count:     200
Overhead clusters:        270
Free blocks:              3716
Free inodes:              989
First block:              1
Block size:               1024
Fragment size:            1024
Reserved GDT blocks:      15
Blocks per group:         8192
Fragments per group:      8192
Inodes per group:         1000
Inode blocks per group:   250
Filesystem created:       Tue Dec 30 11:40:10 2025
Last mount time:          n/a
Last write time:          Tue Dec 30 11:40:10 2025
...
```

### badblocks
This works best for small devices so in the low GiB range.

The following command is destructive as it tests the writing.
```sh
badblocks -wsv /dev/nbd4
```

Output:
```
Checking for bad blocks in read-write mode
From block 0 to 3999
Testing with pattern 0xaa: done
Reading and comparing: done
Testing with pattern 0x55: done
Reading and comparing: done
Testing with pattern 0xff: done
Reading and comparing: done
Testing with pattern 0x00: done
Reading and comparing: done
Pass completed, 0 bad blocks found. (0/0/0 errors)
```

## Bonus

The [Block Block Device][11] project by [williambl][12] was a very nice find
as this post was being typed up. The idea is it is a Minecraft mod and a
[NBDKit][10] where by data iis stored by red stone in a Minecraft world.

The short version of how it works is the Minecraft mod exposes the ability to
be able to query the state of red stone and the plugin communicates with the
mod and exposes it as a network block device.
ss
## Future

* Provide my own storage for the block device that isn't a single file.

[0]: https://en.wikipedia.org/wiki/Network_block_device
[1]: https://www.qemu.org/docs/master/tools/qemu-nbd.html
[2]: https://github.com/NetworkBlockDevice/nbd/blob/master/doc/proto.md
[3]: https://docs.kernel.org/admin-guide/blockdev/nbd.html
[4]: https://github.com/vi/nbdserv
[5]: https://github.com/vi/rust-nbd
[6]: https://www.zerofs.net/
[7]: https://github.com/john-parton/tokio-nbd
[8]: https://github.com/vi/nbdserve/pull/3
[9]: https://github.com/bignaux/lwNBD
[10]: https://gitlab.com/nbdkit/nbdkit
[11]: https://github.com/williambl/blockblockdevice
[12]: https://github.com/williambl
[13]: https://www.youtube.com/watch?v=oiCue9hUN-A
[14]: https://libguestfs.org/guestfish.1.html
[15]: https://libguestfs.org/
