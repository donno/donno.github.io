---
layout: post
title:  Linux for QEMU MicroVM.
date:   2026-05-24 20:00:00 +1030
---

After coming across [smolBSD][smolbsd], which is a project to run a very small
NetBSD kernel with small user space for building MicroVM, I wanted to look
at doing something simliar with Linux. There was two core parts of this,
the first was making a minimal inital RAM filesystem using BusyBox and the
other was compiling the kernel.

While getting this off the ground, I started using an existing Linux kernel
so I could focus on user space, then I built Linux Kernel starting from
minimal configuration and slowly turning things on through trial and error.

For reference, using the Kernel from Ubuntu 26.04 Netboot, it took 8 seconds
to get to /init. A hint if your kernel doesn't output the timings add
`printk.time=t`, compared with the built from source kernel, 0.332004 seconds.

In the end I had a minimal Linux kernel and two initial file system images,
one with Dropbear for SSH access and the other with Valkey to test using it to
host a single application. Both had BusyBox.

## First initial RAM file system
The first iteration with a handwritten script to serve as `init` and it would
install the symlinks for BusyBox's multi-call binary (one binary contains all
commands).

```sh
mkdir busybox-root && cd busybox-root
mkdir -p bin dev etc lib mnt proc sys tmp var home
wget -O bin/busybox https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
cat > init << EOF
#!/bin/busybox sh
/bin/busybox --install /bin
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /tmp
sh
EOF
cat > etc/passwd << EOF
root:x:0:0:root:/root:/bin/sh
nobody:x:65534:65534:nobody:/:/sbin/nologin
EOF
cat > etc/group << EOF
root:x:0:root
nobody:x:65534:
EOF
find . | cpio -ov --format=newc | gzip --best > /host/bb-initramfz && find . | cpio -ov --format=newc > /host/bb-initramfs && echo "Built images"
```

* The passwd and group were set-up as I was trying out  `mdev` and that
  dealt with different users.
* The command at the end rebuilds the images.
* I suspect that this init failed when using the kernel built ourselves because
  the `CONFIG_BINFMT_SCRIPT` option was not provided.. The documentation even says
  > Most systems will not boot if you say M or N here. If unsure, say Y.

## Using BusyBox's init

The piece of this puzzle I was missing and didn't understand was `/etc/inittab`
in the final image.

Essentially instead of commands in the `init` shell script, the commands to run
can be put in the `inittab` under `sysinit` which as the name suggests occurs
when you do the `init`.

```
::sysinit:/bin/mount -a
::sysinit:/bin/hostname -F /etc/hostname
::sysinit:/bin/mkdir -p /dev/pts
::sysinit:/bin/ln -sf pts/ptmx /dev/ptmx
::sysinit:/bin/mount -t devpts -o gid=5,mode=620,ptmxmode=666 devpts /dev/pts
::sysinit:/bin/mkdir -p /dev/shm
::sysinit:/bin/mount -t tmpfs -o mode=1777,nosuid,nodev tmpfs /dev/shm
::sysinit:/bin/ip link set lo up
::ctrlaltdel:/bin/reboot
::shutdown:/bin/echo SHUTTING DOWN
::shutdown:/bin/umount -a -r
tty1::respawn:/bin/getty 38400 tty1
tty2::askfirst:/bin/getty 38400 tty2
tty3::askfirst:/bin/getty 38400 tty3
tty4::askfirst:/bin/getty 38400 tty4
```

* What about mounting `/dev` and `/proc`?
    * That is handled by `/etc/fstab`
      ```
      devtmpfs /dev devtmpfs defaults 0 0
      proc /proc proc defaults 0 0
      ```
    * The question to follow-up is, why isn't `/dev/shm` setup through `fstab`?
* Since this reads `/etc/hostname` that file needs to be created.
  `echo bbmicrovm > etc/hostname`
  * It would be good to be able to set this via a kernel parameter so its
    provided on the command line of the kernel.
* Downside is due to the use of `getty` it now wants to login rather than
  auto-login.
    * This requires setting up `/etc/shadows` with a password.
* To save me having to add an entry every time I wanted to try something else
   out on initalisation, I added `::sysinit:/bin/sh /mystart` such that it
   will call the script `mystart`.

## Various Issues
```
mount: mounting devpts on /dev/pts failed: No such device
```

* The first error was due kernel and  `/etc/inittab` configuration
    * The problematic line of the `inittab` was `::sysinit:/bin/mount -t devpts -o gid=5,mode=620,ptmxmode=666 devpts /dev/pts`
    * This is simply because those the pseudo teletype terminal wasn't
      configured. I didn't write down which kernel configuration this was
      related to.
    * The other `/dev/pts` parts are also removed.
  * Initial error
    `mount: mounting tmpfs on /dev/shm failed: Invalid argument`
  * Call through strace.
    `mount("tmpfs", "/dev/shm", "tmpfs", MS_NOSUID|MS_NODEV|MS_SILENT, NULL) = -1 EINVAL (Invalid argument)`
  * Cause:
    `CONFIG_TMPFS is false but `grep -i tmpfs /proc/filesystems` included tempfs.`
    * Filesystems -> Pseudo filesystems -> Tmpfs virtual memory file system support (former shm fs),
    * "Tmpfs POSIX Access Control Lists " was turned on as wel.

That worked they now mounted.
```
~ # mount
rootfs on / type rootfs (rw,size=246996k,nr_inodes=61749)
devtmpfs on /dev type devtmpfs (rw,relatime,size=246996k,nr_inodes=61749,mode=755)
proc on /proc type proc (rw,relatime)
tmpfs on /run type tmpfs (rw,nosuid,nodev,relatime,size=50336k,mode=755)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,relatime)
devpts on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=666)
```

## Building Kernel

Since my host operating system is Microsoft Windows, I used the Alpine 3.22
distribution I had in WSL2.

The main source for setting-up the kernel was [bluedragon1221][7.1]'s
[minimal Linux project][7.2]. This also served as a great example of what I was
missing from being able to use BusyBox's init instead of my own shell script.

* `apk add alpine-sdk flex bison gawk bc ncurses-dev elfutils-dev git`
    * The other packages are needed for building.
    * `git` was used to clone the repository. A tar could have been used
      instead.
    * `ncurses-dev` is needed for the configuration menu.
    * `elfutils-dev` was needed for `gelf.h`

```sh
git clone --depth 1 --branch "v6.18" https://github.com/torvalds/linux
make tinyconfig
make menuconfig
make -j4
cp arch/x86/boot/bzImage /mnt/d/vms/qemu/linux6.18-virtio-donno-net.bzImage
cp System.map /mnt/d/vms/qemu/linux6.18-virtio-donno-net.System.map
cp .config /mnt/d/vms/qemu/linux6.18-virtio-donno-net.config
```

* With v3.19 (7d0a66e4b) and ended up with a 800kb kernel (bzImage).
* It didn't do the logging on boot.

* [CONFIG_VIRTIO_MMIO][6] - Required due to `microvm` machine type lacking a PCI bus.

## Timings

The initial output didn't include the timing information.
```
NET: Registered PF_INET6 protocol family
Unpacking initramfs...
printk: legacy console [hvc0] enabled
Segment Routing with IPv6
In-situ OAM (IOAM) with IPv6
```

Eventually I stumped across [CONFIG_PRINTK_TIME][3.2] which was turned off
when this kernel was built. however, thee behavior can be controlled by the
[kernel command line parameter][kernel-parameters] `printk.time=1`, so that is
what I used instead, as seen below:
```
[    0.056001] NET: Registered PF_INET6 protocol family
[    0.056001] Unpacking initramfs...
[    0.080003] printk: legacy console [hvc0] enabled
[    0.096004] Segment Routing with IPv6
[    0.096004] In-situ OAM (IOAM) with IPv6
```

## Networking
Turned on the networking stack as seen by
```
NET: Registered PF_NETLINK/PF_ROUTE protocol family
NET: Registered PF_VSOCK protocol family
NET: Registered PF_INET protocol family
```
The latter option corresponds with [CONFIG_INET][10]

* However, still had no `eth0`.
* The cause "Network Device Support" was not enabled
    * This setting was found under the Device Drivers option.
    * The config name for this setting is [CONFIG_NETDEVICES][4]
* Need "Virtio network driver"
    * This is Device Drivers -> Network device support
    * The config name for this setting is [CONFIG_VIRTIO_NET][5]
* I also enabled "Wireguard secure network tunnel" just in case.

The next part was setting up DHCP so in the `/mystart` script added the
following:
```sh
ip link show > /dev/hvc0
udhcpc --interface eth0 --foreground --script /usr/share/udhcpc/default.script > /dev/hvc0
ip link show > /dev/hvc0
```

The result (omitting the ip link as that isn't relevant yet) was didn't work:
```
udhcpc: started, v1.35.0
udhcpc: socket(AF_PACKET,2,8): Address family not supported by protocol
```

* Need "Packet socket"
    * Found Kernel Configuration -> Networking Support -> Networking options
    * The [CONFIG_PACKET][9] configuration corresponds to a module called `af_packet`.

Rerunning it:
```
udhcpc: started, v1.35.0
udhcpc: broadcasting discover
udhcpc: sendto: Network is down
udhcpc: read error: Network is down, reopening socket
```

Fix: `ip link set eth0 up` first.

It still wasn't working, but at least once I ended up with:
```
udhcpc: started, v1.35.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 10.0.2.15, server 10.0.2.2
```

Take step back and try static IP with `ip addr add 192.168.1.100/24 dev eth0`
which resulted in error, which was hard to find, needed to copy a static version
of `strace` into the image.
```
socket(AF_UNIX, SOCK_DGRAM|SOCK_CLOEXEC, 0) = -1 EAFNOSUPPORT (Address family not supported by protocol)
```

* That problem can be fixed by enabling Unix Domain Sockets.
    * In the config it is under Networking support ->  Networking options -> Unix domain sockets

There is the following message output from `dmesg`:
> could not register sysctl

* Need "Sysctl support"
    * This enables /proc/sys
    * > The sysctl interface provides a means of dynamically changing certain kernel parameters and variables on the fly without requiring
    * Found Kernel Configuration -> File systems -> Pseudo filesystems > Sysctl support (/proc/sys)
    * This is the [CONFIG_PROC_SYSCTL][12] configuration option.
    * The warning associated with this is it will add 8KB to the kernel.

Still doesn't work it gets stuck (no IP is selected) and when an IP is selected
it never returns.

**Progress**: discovered `/etc/network/interfaces` is used by `ifup` and the build
of BusyBox that is being used doesn't include that applet so its configuration
is pointless. Switching to a purely manual approach, does improve it but
hits an error:

```sh
ip addr add 10.0.2.15/24 dev eth0
ip route add default via 10.0.2.2
ip link set eth0 up
```

The error is:
> ip: RTNETLINK answers: Network unreachable

The fix is to set-up the link first.
```sh
ip link set eth0 up
ip addr add 10.0.2.15/24 dev eth0
ip route add default via 10.0.2.2
```

Now there is no errors and:
```sh
ip addr show eth0 2>&1 > /dev/hvc0
```

Outputs:
```
4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe12:3456/64 scope link tentative
       valid_lft forever preferred_lft forever
```

Next problem is the networking is hanging, `wget` and `nslookup`.

This is due to the Windows Hypervisor Platform (WHPX) having issues
delivering interrupts due to `pic=on` on Microsoft Windows 10 as per
the [QEMU documentation][15]. The fix is to turn on [CONFIG_KVM_GUEST][14] as
the documentation from 6.18.13 states:
>  It includes a paravirtualized clock, so that instead of relying on a PIT
   (or probably other) emulation by the underlying device model, the host
provides the guest with timing infrastructure such as time of day, and system time

This setting was under Processor type and features -> Linux guest support >
 >KVM Guest support (including kvmclock). This in turn meant turning on
[CONFIG_HYPERVISOR_GUEST][13] which enables basic hypervisor detection and
platform setup.

The expectation if this works is `cp /proc/interrupts /dev/hvc0` should show
show IO-APIC next to the interrupt for the VirtIO network device. However, in my case
it still stays XT-PIC, however after playing around it ended up saying IO-APIC.
That said, when I turn off `pic` and `pit` on QEMU, there is no output
from QEMU after it says the accelerator is operational

The following should be in the output of `dmesg`:
> IOAPIC[0]: apic_id 2, version 32, address 0xfec00000, GSI 0-23

**Before**
```
  0:         29   XT-PIC      timer
  2:          0   XT-PIC      cascade
 11:          2   XT-PIC      virtio1
 12:         15   XT-PIC      virtio0
 ```
**After**
Removing `noapic nolapic acpi=off` from the kernel command line, the last
option was doing nothing as it is being ignored anyway.
```
  0:         66  IO-APIC   2-edge      timer
  2:          0   XT-PIC      cascade
 11:          2  IO-APIC  11-edge      virtio1
 12:         18  IO-APIC  12-edge      virtio0
```

Now check the network is working by adding the following to the `mystart.sh`
```
wget http://ifconfig.me/all -O -
```

## Set-up SSH Server
For this, the plan is to use [`dropbear`][dropbear] which can be statically
compiled.
```sh
wget -O bin/dropbear https://static-binaries.gitlab.io/dropbear/dropbear-2019.78.x86_64-linux-android
chmod +x bin/dropbear
```

Trying to run it in the foreground to ensure we have everything working:
* Added command `/bin/dropbear -F -R 2>&1 > /dev/hvc0` to `/mystart.sh`
* > setsockopt(4, SOL_IPV6, IPV6_TCLASS, [16], 4) = -1 ENOPROTOOPT (Protocol not available)
* > openat(AT_FDCWD, "/var/run/dropbear.pid", O_WRONLY|O_CREAT|O_TRUNC, 0666) = -1 ENOENT (No such file or directory)
  * [`/var/run`][rhs-varrun] is for run-time variable data and has since been
    moved to `/run` but Dropbear seems to be following the old ways.
    * It is valid to implement /var/run as a symlink to /run.
  * Due to the folder not existing it means it is unable to write the pid file
    so it won't be able to detect if it is already running.
* The server keys are not created until the first user tries to connect.
  * This requires `/etc/dropbear/` to exist.
* Can't SSH as root with a password as that is disallowed by default, so the
  set-up a normal user account for that.
* This is resulting in
  > Login attempt for nonexistent user
  * The user can be logged in interactive.*
  * Log message comes here: https://github.com/mkj/dropbear/blob/672f5963a525c167dc6c103b86d2aa1aceb1a3ec/src/svr-auth.c#L262
  * Confirming permissions:
    ```
    drwxr-xr-x    3 root     root           0 May 12  2026 .
    drwxr-xr-x   14 root     root           0 Jan  1 00:00 ..
    drwxr-xr-x    3 donno    users          0 May 12  2026 donno
    total 0
    drwxr-xr-x    3 donno    users          0 May 12  2026 .
    drwxr-xr-x    3 root     root           0 May 12  2026 ..
    drwx------    2 donno    users          0 May 12  2026 .ssh
    total 4K
    drwx------    2 donno    users          0 May 12  2026 .
    drwxr-xr-x    3 donno    users          0 May 12  2026 ..
    -rw-------    1 donno    users        462 May 12  2026 authorized_keys
    ```
  * Tried adding /etc/shell.

I took a break here and did the Valey part.
The problem seemed to be with the version of dropbear I was using.
Updating to version 2026.91, gave progress.

```
[53] Jan 01 00:00:00 Not backgrounding
[54] Jan 01 00:00:08 Child connection from 10.0.2.2:50289
[54] Jan 01 00:00:08 Generated hostkey /etc/dropbear/dropbear_ecdsa_host_key, fingerprint is SHA256:hyX6N9so323/GHx7zhbdRUJYcdRoPimsNAChDQk8acI
[54] Jan 01 00:00:09 Pubkey auth succeeded for 'donno' with ssh-rsa key SHA256:sfHZhHKkXNMnspPTa2eZSkZ2/XPdYEGUeuoUfvtjFlA from 10.0.2.2:50289
[54] Jan 01 00:00:09 pty_allocate: openpty: No such file or directory
[54] Jan 01 00:00:09 No pty was allocated, couldn't execute
[54] Jan 01 00:00:09 Exit (donno) from <10.0.2.2:50289>: Error reading: Connection reset by peer
```

* The INSTALL.md for dropbear happens to metion about `openpty`.
  > If `openpty()` is being used (`HAVE_OPENPTY` defined in `config.h`) and it fails, you can try compiling with `--disable-openpty`.
* I suspect this may need CONFIG_UNIX98_PTYS from Device Drivers -> Character devices -> Unix98 PTY support, however
that option seemed to have disappeared after I disabled the expert mode and
the setting is enabled on the config.
* The problem was there is no `/dev/pts`, needed to add
  `mount -t devpts devpts /dev/pts -o gid=5,mode=620`.

Connecting and disconnecting:
```
[58] Jan 01 00:00:06 lastlog_perform_login: Couldn't stat /var/log/lastlog: No such file or directory
[58] Jan 01 00:00:06 lastlog_openseek: /var/log/lastlog is not a file or directory!
[58] Jan 01 00:00:06 wtmp_write: problem writing /dev/null/wtmp: Not a directory
[57] Jan 01 00:00:32 wtmp_write: problem writing /dev/null/wtmp: Not a directory
[57] Jan 01 00:00:32 Exit (donno) from <10.0.2.2:51233>: Disconnect received
```

* When building Dropbear there is an option to disable lastlog
  (`--disable-lastlog`) which could be the simpler option. The idea of this VM
  was short lived / not persistent so the log wouldn't be all taht useful anyway.
* When building Dropbear there is an option to disable wtemp (`--disable-wtmp`)
  which is meant to also store uuser logins, logouts, system boots, and
  shutdowns. To view them you can type `last` in a shell.

The final result was the initial RAM filesystem came to 2,422,800 bytes
uncompressed containing and the Linux kernel was 2,892,800 bytes:
* Busybox
* Dropbear
* Several small text files

For Linux there were some kernel models that are unused:
* Wireguard
* i8042: No controller found

## Valkey
Instead of hosting SSH considered what if the plan was to run an appliance.

In this case, an existing container image will be used
```sh
wget https://github.com/opencontainers/umoci/releases/download/v0.6.0/umoci.linux.amd64
wget https://github.com/lework/skopeo-binary/releases/download/v1.20.0/skopeo-linux-amd64
chmod +x umoci.linux.amd64 skopeo-linux-amd64
./skopeo-linux-amd64 --insecure-policy copy docker://docker.io/valkey/valkey:9.0.0 oci:valkey:9.0.0
./umoci.linux.amd64 unpack --image valkey:9.0.0 /busybox-root/opt/valkey
```

To the `/mystart.sh` script added:
```sh
echo Running valkey > /dev/hvc0
chroot /opt/valkey/rootfs ./usr/local/bin/valkey-server 2>&1 > /dev/hvc0
reboot
```

Re-built the image and ran it.

```
[    0.980043]     TERM=linux
Hello from mystart.sh
Running valkey
The futex facility returned an unexpected error code.
Aborted
```

* The much larger initial RAM file system has made it take quite a bit longer
  to boot.
* The program fails as when compiling the kernel, the futex feature wasn't
  enabled.
    * Found under  Linux Kernel Configuration -> General setup ->
      Configure standard kernel features (expert users) -> Enable futex support
    * [CONFIG_FUTEX][17].
    * This was hard to find, I then discovered you can search in menuconfig by
      typing / and a search box appears.
* If this did work, then would added `hostfwd=tcp::2229-:6379` to map 6379
  within the VM to port 2229 on the host.

next one:
```
46:M 01 Jan 1970 00:00:01.180 * oO0OoO0OoO0Oo Valkey is starting oO0OoO0OoO0Oo
46:M 01 Jan 1970 00:00:01.184 * Valkey version=9.0.0, bits=64, commit=00000000, modified=0, pid=46, just started
46:M 01 Jan 1970 00:00:01.184 # Warning: no config file specified, using the default config. In order to specify a config file use ./usr/local/bin/valkey-server /path/to/valkey.conf
46:M 01 Jan 1970 00:00:01.184 * Increased maximum number of open files to 10032 (it was originally set to 1024).
46:M 01 Jan 1970 00:00:01.184 * monotonic clock: POSIX clock_gettime
46:M 01 Jan 1970 00:00:01.184 # Failed creating the event loop. Error message: 'Function not implemented'
```

Where the function not implemented is:
```
epoll_create(1024)                      = -1 ENOSYS (Function not implemented)
```

After sorting out futex wasn't enabled and now this one I realised it was
because an expert user section was enabled, rather than try to find the config
that enables it, I simply went back and turned off the expert user setting
which keeps Futex support enabled (as turning the expert setting on essentially
turns of Futex).

```
Hello from mystart.sh
Running valkey
53:M 01 Jan 1970 00:00:01.040 * oO0OoO0OoO0Oo Valkey is starting oO0OoO0OoO0Oo
53:M 01 Jan 1970 00:00:01.040 * Valkey version=9.0.0, bits=64, commit=00000000, modified=0, pid=53, just started
53:M 01 Jan 1970 00:00:01.040 # Warning: no config file specified, using the default config. In order to specify a config file use ./usr/local/bin/valkey-server /path/to/valkey.conf
53:M 01 Jan 1970 00:00:01.040 * Increased maximum number of open files to 10032 (it was originally set to 1024).
53:M 01 Jan 1970 00:00:01.040 * monotonic clock: POSIX clock_gettime
                .+^+.
            .+#########+.
        .+########+########+.           Valkey 9.0.0 (00000000/0) 64 bit
    .+########+'     '+########+.
 .########+'     .+.     '+########.    Running in standalone mode
 |####+'     .+#######+.     '+####|    Port: 6379
 |###|   .+###############+.   |###|    PID: 53
 |###|   |#####*'' ''*#####|   |###|
 |###|   |####'  .-.  '####|   |###|
 |###|   |###(  (@@@)  )###|   |###|          https://valkey.io
 |###|   |####.  '-'  .####|   |###|
 |###|   |#####*.   .*#####|   |###|
 |###|   '+#####|   |#####+'   |###|
 |####+.     +##|   |#+'     .+####|
 '#######+   |##|        .+########'
    '+###|   |##|    .+########+'
        '|   |####+########+'
             +#########+'
                '+v+'

53:M 01 Jan 1970 00:00:01.044 * Server initialized
53:M 01 Jan 1970 00:00:01.044 * Ready to accept connections tcp
```

I was then able to connect on `valkey-cli.exe -p 2229`.

The last time from `printk` was 0.988045 seconds and given Valkey is outputting
the same time with each message it seems to suggest the actual kernel
start time to was 1.04 seconds.

## Future Kernel Options

Additional features that I would consider enabling in the future are:
* File system support
    * Provide a place for persistent storage
* Container support (or container kernel variant)
    * Namespace
    * Control groups
* Linux guest support
* Link Time Optimization (LTO)

## Initial file system

Outside of busybox which populates `/bin`

The folder/file list looks like so
```
./bin/busybox
./bin/... (symlinks to busybox here)
./dev
./root
./sbin
./sbin/init
./etc
./etc/fstab
./etc/hostname
./etc/inittab
./etc/shadow
./etc/environment
./etc/passwd
./etc/group
./init.old
./init
./mystart
./proc
./usr
./usr/sbin
```

Optionally a `/etc/motd` could be provided which will be shown when a user
logins.

## Future
Where to go next from this:
- Explore the idea of using EROFS - (Enhanced Read-Only File System) for either
  the root file system or additional appliances as well as maybe Overlay FS for
  providing write-layer.
- Build a `ext3` filesystem image of the contents of the container so it can be
  provided to system separated and mounted to `/app_root`.
- Going too far into containers, i.e. enable cgroups and namespaces to allow
  container to work such that `runc` does as per my previous experiment.
  However, at this point then this is trying to reinvent `boot2container`.
  * It might be nice to explore setting up a Linux kernel targetting microvm
    for use by boot2container. There is already a minimised for QEMU
    [kernel][21] produced but haven't checked if it works with `microvm`.
* Consider building own busybox - that could have saved a lot of effort with
  the lack of `ifup`. It may be needed for the `hardshutdown` applet to cause
  the VM to stop - this wasn't needed in the end enabling the APIC allowed
  reboot to do the triple fault.
  * Alternative is try to use `sysctl kernel.reboot=1"` since I turned on the
   `sysctl` config. See https://docs.kernel.org/admin-guide/sysctl/kernel.html
* Build BusyBox and Dropbear from source so there is no question about GPL
  conformance.
    * I had to build Dropbear from source as the pre-built version didn't
      work.
* Consider addressing the following seen when running `dropbear` (SSH server).
  > setsockopt(4, SOL_IPV6, IPV6_TCLASS, [16], 4) = -1 ENOPROTOOPT (Protocol not available)
* Write own `/init` and reduce files

## Sources
The sources for the kernel configuration and scripts for building the initial
root filesystems mentioned in this post are available from
[https://github.com/donno/linux-microvm](https://github.com/donno/linux-microvm)

[0]: https://www.qemu.org/
[1]: https://www.qemu.org/docs/master/system/i386/microvm.html
[2]: https://www.kernel.org/doc/html/latest/filesystems/ramfs-rootfs-initramfs.html#what-is-initramfs
[3.1]: https://lwn.net/Articles/827334/I
[3.2]: https://www.kernelconfig.io/CONFIG_PRINTK_TIME
[4]: https://www.kernelconfig.io/CONFIG_NETDEVICES
[5]: https://www.kernelconfig.io/CONFIG_VIRTIO_NET
[6]: https://www.kernelconfig.io/CONFIG_VIRTIO_MMIO
[7.1]: https://github.com/bluedragon1221
[7.2]: https://github.com/bluedragon1221/minlinux2
[8]: https://www.kernelconfig.io/CONFIG_EARLY_PRINTK
[9]: https://www.kernelconfig.io/CONFIG_PACKET
[10]: https://www.kernelconfig.io/CONFIG_INET
[11]: https://www.kernelconfig.io/CONFIG_UNIX
[12]: https://www.kernelconfig.io/CONFIG_PROC_SYSCTL
[13]: https://www.kernelconfig.io/CONFIG_HYPERVISOR_GUEST
[14]: https://www.kernelconfig.io/CONFIG_KVM_GUEST?q=&kernelversion=6.18.23&arch=x86
[15]: https://www.qemu.org/docs/master/system/whpx.html
[16]: https://www.kernelconfig.io/CONFIG_X86_MPPARSE
[17]: https://www.kernelconfig.io/CONFIG_FUTEX
[20]: https://blinry.org/tiny-linux/
[21]: https://gitlab.freedesktop.org/gfx-ci/boot2container/-/releases/v0.10.0/downloads/linux-x86_64-qemu

[rhs-varrun]: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s13.html
[kernel-parameters]: https://www.kernel.org/doc/Documentation/admin-guide/kernel-parameters.rst
[dropbear]: https://matt.ucc.asn.au/dropbear/dropbear.html
[smolbsd]: https://github.com/NetBSDfr/smolBSD
