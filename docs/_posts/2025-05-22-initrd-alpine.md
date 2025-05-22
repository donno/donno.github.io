---
layout: post
title:  "initrd for Alpine"
date:   2025-05-22 23:00:00 +1030
---

This is a follow-up to the weekend's project of playing with the
[crosvm project][0] (a virtual machine monitor). This time the focus is on
building a initial RAM disk for Alpine.

## Original plan

Lets start by addressing why focus on the RAM disk and not a hard drive image.
This mostly came down to most of the tools being very difficult to run
inside a container or WSL2. This is because the two most common options both
require mounting the image which required kernel modules not available. The
two ways being via a loop device (`losetup`) or a NBD (network block device).
The latter option is what the [`alpine-make-vm-image`](1) project uses.

> Attaching image alpine.qcow2 as a NBD device
modprobe: can't change directory to '/lib/modules': No such file or directory
ERROR: No available nbd device found!

The other option I tried was creating a VHD (the format used by Windows) which
can be mounted bare to WSL. This was because I would likely end up using the
resulting image with Hyper-V anyway so a Windows-specific option was an
acceptable compromise.

Within PowerShell
```powershell
$vhd = New-VHD -Path alpine.vhdx -SizeBytes 100MB
wsl --mount --vhd $vhd.path --bare
```

Within WSL
```sh
$ lsblk
sde   8:64   0   100M  0 disk
```

This looked promising but it didn't get future than that.

## Root filesystem - Initial Attempt
Rather than start with the pre-made minimal root filesystem for Alpine, I again
wanted to go with installing the system from apk. This was building on the
experience from .

Started a new Alpine container then within it got to work:
```sh
mkdir /rootfs
cd /rootfs
curl -LO https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.6/x86_64/apk.static
cp /etc/resolv.conf rootfs/etc/
chroot .
./apk.static --arch x86_64 -X https://dl-cdn.alpinelinux.org/alpine/edge/main/ --root . --initdb --no-cache --allow-untrusted add alpine-conf coreutils alpine-base
```

In this case it was executing `apk` with the new root and had issues resolving
domain names and thus downloading packages. I'm writing this a two days after
this part of the experience and it has me wondering why do it this way. The
easier thing would have been to keep `apk.static` outside `/rootfs` and
install into the folder as it can do that. This already proved to work from
when creating the [barebone containers](5).

## mkinitfs - Second attempt
Next time, I tackled it differently and this time by running apk outside.
As everything was being done within an Alpine container anyway simply used the
pre-installed `apk`.

Thus started with this.
```sh
apk --arch x86_64 -Xhttps://dl-cdn.alpinelinux.org/alpine/latest-stable/main/ --root /rootfs --initdb --no-cache --allow-untrusted add alpine-base
```

Next-up was creating the initial filesystem from it using [`mkinitfs`](2). The
first problem was it needs to know the kernel version.

TO determine the kernel version I first download it:
```sh
wget https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/netboot-3.21.3/vmlinuz-virt
file vmlinuz-virt
```
Within the output this has the version which is: `6.12.13-0-virt`.

The complete command line is:
```sh
apk add mkinitfs
mkinitfs -c /etc/mkinitfs/mkinitfs.conf -b /rootfs/ -o /work/initramfs-virt 6.12.13-0-virt
```

Trying to boot with this fails quite quickly
```
[    2.873264] Mounting boot media...
 * Mounting boot media: /init: line 786: nlplug-findfs: not found
[    2.885078] Mounting boot media: failed.
```

The problem is the `init` script it uses has a program that is missing. The
the init file to use can be overwritten with the `-i` option. Try to use
the normal `init` program from `/sbin` instead.
```sh
mkinitfs -c /etc/mkinitfs/mkinitfs.conf -b /rootfs/ -o /work/initramfs-virt -i /rootfs/sbin/init 6.12.13-0-virt
```

This however fails, with the errors:
```
can't run '/etc/init.d/rcS': No such file or directory
can't open /dev/tty2: No such file or directory
can't open /dev/tty3: No such file or directory
can't open /dev/tty4: No such file or directory
```

It important to add that when writing up this post, the /rootfs directory that
was made into a working initial file system was retried with the above command
and it still failed.

One good thing that came out of this was extracting the modules from
`/lib/modules` from the netboot's `initramfs-virt` file which is used later.

## Third attempt

I was confident that the populating of the system was working fine and it was
mainly with the conversion of the folder into the single file that was the
problem. This is where I came across a Gist titled
[run a minimal alpine based initramfs in VM machine](3). This has the command
for converting from the extracted minimal root filesystem to the file.

The key part is the tool `cpio` which is a file archiver from the early Unix
days. I ended up checking the [firecracker-initrd](4) project for how it sets
it up and sure enough it uses that tool but the key part I hadn't done yet
was set-up the `/etc/init.d`.

```sh
# Install the packages as above with apk
...

# Configure startup
cp /rootfs/sbin/init /rootfs/init
echo "ttyS0" >> /rootfs/etc/securetty
ln -sf /etc/init.d/devfs  /rootfs/etc/runlevels/boot/devfs
ln -sf /etc/init.d/procfs /rootfs/etc/runlevels/boot/procfs
ln -sf /etc/init.d/sysfs  /rootfs/etc/runlevels/boot/sysfs
ln -sf agetty             /rootfs/etc/init.d/agetty.ttyS0
ln -sf /etc/init.d/agetty.ttyS0 /rootfs/etc/runlevels/default/agetty.ttyS0

# Create the initial file system image.
(cd /rootfs && find . -print0 | cpio --null --create --verbose --format=newc > /work/initrdfs && cd - >/dev/null;)
```

With this I was able to get get a bootable image that didn't error out the same
as the one from `mkinitfs` did. It had the standard OpenRC set-up that is found
in a typical Alpine distribution.

Running it with:
```sh
qemu-system-x86_64 -m 512  -kernel vmlinuz-virt -initrd initrdfs
```

## Networking
The next challenge was networking, which the first step was add `-net nic -net user`
to the QEMU command line to add networking.

The first thing wanted was `/sys/class/net` to show `eth0`,

From there the approach taken was see about setting it up with Alpine's scripts
and seeing how we go from there.

1. Starting with `ifup` that states `ifup: could not parse /etc/network/interfaces`
2. Provide a basic `/etc/network/interfaces`:
     ```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual
     ```
3. `ifup` doesn't complain and `ifup lo` is fine but `ifup eth0` states
  ```
  Cannot find device "eth0"
  Device "eth0 does not exist."
  ```
4. Check `/sys/class/net` which only contains `lo`.
5. Stumble across `setup-devd` for detecting devices and try using `mdev`.
6. When it asks to Scan the hardware to populate `/dev`, say yes.
7. It doesn't find anything. This is where I suspect it needs the modules.
8. Copy the `/lib/modules/` folder from an existing `initrd-virt` into
  `/rootfs/lib/modules`
9. Run `setup-devd mdev`
   ```
 (none):~# setup-devd mdev
 * Starting busybox mdev ...                                              [ ok ]
 * Scanning hardware for mdev ...                                         [ ok ]
 * Loading hardware drivers ...
[   28.718007] e1000: Intel(R) PRO/1000 Network Driver
[   28.720689] e1000: Copyright (c) 1999-2006 Intel Corporation.
[   28.733122] ACPI: \_SB_.LNKC: Enabled at IRQ 11
[   29.058471] e1000 0000:00:03.0 eth0: (PCI:33MHz:32-bit) 52:54:00:12:34:56
[   29.063053] e1000 0000:00:03.0 eth0: Intel(R) PRO/1000 Network Connect [ ok ]
(none):~# ls /sys/class/net/
eth0  lo
(none):~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
     ```
   Now that is promising.
10. Configure the interface using `setup-interfaces` and choosing `dhcp`.
    `udhcpc` should start and select an IP. In the case of QEMU, which ahs a
    DHCP server used by the guests, it will also populate the name server so
    you can resolve IPs.
11. At this point networking is working and all the pieces of the puzzle are
   there and now it is a matter of untangling and getting them put together.

### Pre-configured
The next step is setting up the image so the manual commands ran above are
handled automatically.

This is where pealing back the curtain on  `setup-devd mdev` reveals that the
important part is:
```sh
rc-update add hwdrivers sysinit
rc-update add mdev sysinit
```

Of course, in the previous step the important part was the fact it ran
`rc-service start` on those services to start them right away which is how there
was immediate feedback without having to restart the system, but for
pre-configuring they only need to be added.

The approach taken was to simply add it like so:
```sh
  chroot /rootfs /bin/sh -c 'rc-update add hwdrivers sysinit'
  chroot /rootfs /bin/sh -c 'rc-update add mdev sysinit'
```

That could likely be replaced with the same approach as the other init.d set-up
as seen in `firecracker-initrd`, which would look liek this:
```sh
ln -sf /etc/init.d/hwdrivers /rootfs/etc/runlevels/sysinit/
ln -sf /etc/init.d/mdev /rootfs/etc/runlevels/sysinit/
```
This creates a soft-link within the second path to the former. The thing to
realise is the link is written to point to the first path given verbatim. That
confused me as first as it looks like it would link it to the file on the
current machine. It not until you boot the system where it ends up resolving.
From the perspective of build-time, it is essentially
`/rootfs/etc/init.d/hwdrivers` that it is referring to.

The reset of the networking for example starting `sshd` when `eth0` comes up
and running `/etc/init.d/networking` is done using the links.

The interfaces file is populated similar to step 2 above ahead of time but
instead of `manual` use `dhcp`.

## Conclusion
The goal of scripting the creating of an initial RAM disk that can be used to
boot Alpine in a virtual machine with networking was achieved.

The scripts created for this post is at [this gist](6).

## Next
* Replace the `chroot` usage with the linking as done for hte other services.
* Unwravel `alpine-base` package into its parts.
* Possibly extend the script to create user accounts and set-up their
  authorised key.
* Make the hostname configurable.

[0]: https://crosvm.dev/book/
[1]: https://github.com/alpinelinux/alpine-make-vm-image
[2]: https://gitlab.alpinelinux.org/alpine/mkinitfs
[3]: https://gist.github.com/gdamjan/1f260b58eb9fb1ba62d2234958582405
[4]: https://github.com/marcov/firecracker-initrd/blob/32930133c2d1e6d3d7e2ec9849542b3ac9b5c61f/container/build-initrd-in-ctr.sh
[5]: {% post_url 2025-04-18-barebone-alpine-containers %}
[6]: https://gist.github.com/donno/c418e1e594af9c0f83df9ad0cc5f20da
