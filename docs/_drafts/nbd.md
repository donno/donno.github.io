
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

## Bonus

The [Block Block Device][11] project by [williambl][12] was a very nice find
as this post was being typed up. The idea is it is a Minecraft mod and a
[NBDKit][10] where by data iis stored by redstone in a Minecraft world.

The short version of how it works is the Minecraft mod exposes the ability to
be able to query the state of redstone and the plugin communicates with the
mod and exposes it as a network block device.

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