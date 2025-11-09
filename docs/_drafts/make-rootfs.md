
This weekend's little project was creating a root file system image. The idea
is to create an image that is bigger than what would be supported by the
initrd.

## Set-up
This project was performed from Alpine Linux.

```sh
apk add e2fsprogs
```

* The `e2fsprogs` package provides the `mkfs.ext4` command for creating an
  `ext4` file system.

If the host or container is not Alpine then download the statically linked
version of `apk` (Alpine Package Keeper) called `apk.static` and use/try that.
```sh
wget https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.10/x86_64/apk.static
chmod +x apk.static
```

## Preparing root fs

The root file system will be prepared in a directory called `root-source`.

### Using Alpine

```sh
apk --allow-untrusted -X https://dl-cdn.alpinelinux.org/alpine/latest-stable/main -U --allow-untrusted --root root-source --initdb add alpine-base
```

A problem with this approach at this time is the resulting image won't be
bootable. There is no boot loader installed and no provision for EFI either.
This makes this only suitable for the direct boot option. Thus this requires
a virtualisation platform where you can boot the kernel image directly.
[Firecracker](1), [crosvm](3) and [QEMU](2) can all do this.

I would like to come back to this another day as the main motivation for using
[initrd](0) last time was to side-step this problem.

### Using Wolfi from Chainguard

_Spoiler_: I didn't end-up with a working system using this approach. This
is likely to be expected as Wolfi is meant for containers only unlike Alpine.

For this case lets start with the initial problems encountered during
the first few attempts at getting this working.

The first was Wolf lacks an `init` system so it fails. To overcome this the
following command was added `ln -sf /usr/bin/busybox root-source/usr/bin/init`
to instruct it to use `busybox` as the `init` system.

However, it failed with:
```
[    5.063946] traps: sh[1] trap invalid opcode ip:723d6d5180d0 sp:7ffda96f5400 error:0 in ld-linux-x86-64.so.2[200d0,723d6d4f9000+29000]
[    5.075462] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

The issue here ended up being the same as a problem I experienced later on
when trying out bootc for the first time. The issue was QEMU was not configured
to allow the `x86-64-v2` micro-architecture to be used. As such adding
`-cpu max` to the command line of `qemu-system-x86_64` overcomes that problem.

Trying again resulted in a different error:

```
[    5.089075] Run /sbin/init as init process
init: applet not found
[    5.287827] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00
[    5.293394] CPU: 0 UID: 0 PID: 1 Comm: init Not tainted 6.17.0-5-generic #5-Ubuntu PREEMPT(voluntary)
[    5.297009] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.17.0-0-gb52ca86e094d-prebuilt.qemu.org 04/01/2014
```

This problem is that the  `init` applet (command) isn't enabled in Wolfi's
version of `busybox` unlike Alpine's.  So the original approach of linking
`busybox` as `init` is not going to work.

Next up try OpenRC and sure enough that worked:
```
[    4.634012] Run /sbin/init as init process
OpenRC init version 0.63 starting
Starting sysinit runlevel

   OpenRC 0.63 is starting up Linux 6.17.0-5-generic (x86_64)
```

Next set of issues:
* `/etc/fstab does not exist`
* `/usr/lib/rc/sh/openrc-run.sh: line 64: hwclock: not found`
    * This is include in the `util-linux-misc` package.
* `/usr/lib/rc/sh/openrc-run.sh: line 96: fsck: not found`
    * This is include in the `util-linux-misc` package.
* `/usr/lib/rc/sh/openrc-run.sh: line 149: umount: not found`
  ** This is include in the `umount` package.
* `/usr/lib/rc/sh/openrc-run.sh: line 39: find: not found`
  * This is included in the `findutils` package.
* `/usr/lib/rc/sh/openrc-run.sh: line 58: eend: not found`
* `/usr/lib/rc/sh/openrc-run.sh: line 211: checkpath: not found`
  * This is included in the `openrc` package as `/usr/lib/rc/bin/checkpath`
* `/usr/lib/rc/sh/openrc-run.sh: line 156: grep: not found`
  * This is included in the `grep` package.
* `/usr/lib/rc/sh/openrc-run.sh: line 28: modprobe: not found`
  * This is included in the `kmod` package.

I never did get this system properly booting.

```sh
apk --arch x86_64 -X https://apk.cgr.dev/chainguard/ -U --allow-untrusted --root root-source  --timeout 60 --initdb add wolfi-base chainguard-keys mount umount openrc util-linux-misc findutils grep
ln -sf /sbin/openrc-init root-source/usr/bin/init
```

* `openrc` for the init system.
* `util-linux-misc` for `hwclock` and `fsck` used by `openrc`
* `mount` for mounting file-systems.
* Several packages were needed for the `openrc-run.sh` script, which were
  mentioned above.

## Create the image
```sh
truncate -s 1G our.rootfs.ext4
mkfs.ext4 -d root-source -F our.rootfs.ext4
```

## Testing with QEMU

```sh
curl --output ubuntu-netboot-kernel http://mirror.aarnet.edu.au/pub/ubuntu/releases/25.10/netboot/amd64/linux
qemu-system-x86_64 -m 512  -kernel ubuntu-netboot-kernel -drive if=virtio,format=raw,file=our.rootfs.ext4 -append "root=/dev/vda"
```

* On Windows add ` -accel whpx` to use the Windows Hypervisor Platform.
  * The difference this makes is
    `[    1.373455] Run /sbin/init as init process`
    Verses
    `[    5.363164] Run /sbin/init as init process`

* The kernel used is from Ubuntu Netboot because the `linux-virt` kernel by Alpine doesn't include support for ext4.
    * It supports squashfs but not sure that can be provided via a block device (i.e. vda)
    * This was discoverable by `ls /lib/modules/6.12.38-0-virt/kernels/fs`
    * It had fat, fuse, nls, overlayfs and squashfs

## Testing with Firecracker

### Configuration

```json
{
  "boot-source": {
    "kernel_image_path": "vmlinux",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off ip=10.200.0.2::10.200.0.1:255.255.255.255::eth0:off"
  },
 "drives": [
    {
      "drive_id": "rootfs",
      "partuuid": null,
      "is_root_device": true,
      "cache_type": "Unsafe",
      "is_read_only": false,
      "path_on_host": "our.rootfs.ext4",
      "io_engine": "Sync",
      "rate_limiter": null,
      "socket": null
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 2048,
    "smt": false,
    "track_dirty_pages": false,
    "huge_pages": "None"
  },
  "cpu-config": null,
  "balloon": null,
  "network-interfaces": [
    {
      "iface_id": "net1",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "crosvm_tap"
    }
  ],
  "vsock": null,
  "logger": null,
  "metrics": null,
  "mmds-config": null,
  "entropy": null
}
```

Using 1024 MiB of memory wasn't enough hand resulted in the error
     `Failure during vcpu run: Out of memory (os error 12)`

### Running
```
rm /tmp/fire-alpine.socket && firecracker --config-file alpine-rootfs.config --api-sock /tmp/fire-alpine.socket
```

Shutting down
```sh
curl --unix-socket /tmp/fire-alpine.socket -X PUT 'http://localhost/actions' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{ "action_type": "SendCtrlAltDel" }'
```

### Problems

* Didn't set-up root user.
  ```chroot /rootfs /bin/sh -c 'adduser root; echo -e "root\nroot" | passwd root```
* Would probably be worth using the alpine-initrd / `mkramfs.sh` as a base such
  that it supports setting-up extra users and configuring ssh access.

## Build from container

This was a little experiment I wanted to try.

For this we will try:
* valkey/valkey:8.1.4-alpine3.22 as the container
* umoci  as the tool
* skopeo for downloading image - umoci doesn't have built-iun support for
  getting an image

```sh
apk add umoci skopeo
skopeo copy docker://valkey/valkey:8.1.4-alpine3.22 oci:valkey:8.1.4-alpine3.22
umoci raw unpack --image valkey:8.1.4-alpine3.22 root-source
truncate -s 1G valkey.rootfs.ext4
mkfs.ext4 -d root-source -F valkey.rootfs.ext4
```

Problems
* No `/sbin/openrc`
* No suitable `/sbin/init` which is configure to run the container.

Solutions
* Write a suitable init that uses the original container information. to start
* Create root-fs from Alpine with runc, convert image so its runnable with
  runc an then set-up the init to use runc on start-up.
  - Shame there is not a quadlet generator for OpenRC like there is for
    systemd.
* Don't reinvent this and instead look to `bootc` - however that is different
  approach then what I was thinking, as that is how to provision/create a
  host system using a `Containerfile`. I'm interested in taking an existing
  container and hosting it.

As such you could think of this as being either a second partition or second
drive and image.

## BootC

The [`bootc` project](5) is working the mission of [Bootable Container Images](6).
In short, its about providinng operationg system updates using OCI container
images and reusing the tooling around containers for building images.

### Features
- Immutable OS
- Readonly at runtime
    - /etc
    - /var
- Writable at build time.
- Supported are Fedora, CentOS Stream, RHEL

### Experiment
I created a image using the "Bootable Container" extension for Podman Desktop.
This means the commands used to turn the the httpd example container into an
image aren't provided here. I also missed the part where you can provide
username and password.

To run the image with QEMU, it looked like this:
```sh
qemu-system-x86_64 -m 1G -drive if=virtio,format=raw,file=bootc-images\image\disk.raw -nographic -cpu max -net nic -net user,hostfwd=tcp::4080-:80
```
Confirm it works by visiting http://localhost:4080/

The `-cpu max` part was to ensure it supported `x86-64-v2`, as when running it
without it it would, fail on start-up, as seen below.
```
[    1.792752] Run /init as init process
Fatal glibc error: CPU does not support x86-64-v2
[    1.799499] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00
```

The `-cpu max` option was not compatible with the `whpx` accelerator, as it
would result in:
```
whpx: injection failed, MSI (0, 0) delivery: 0, dest_mode: 0, trigger mode: 0, vector: 0, lost (c0350005)
qemu-system-x86_64.exe: WHPX: Unexpected VP exit code 4
```

As a result the start time for the image was about 90 seconds which I believe
attributed to not being able to use the visualisation acceleration option.

Overall neat, however my interest is not so much building a image for a new
(virtual) machine from an OCI container image but for taking an existing OCI
container image and baking it into a hard drive image and running it. Ideally,
without the full container infrastructure (i.e. containerd, Podman, Docker).

I would be keen to revisit it for building Alpine images however it doesn't
seem that is possible and from looking a little into it it seems the main thing
that `bootc` utilises is [`ostree`](7) and I'm not sure how compatible Alpine is with
that. Given my original project was to investigate building pre-built Alpine
images `bootc` for that does seem nice as it brings the familiarity of a
`Containerfile` to the process. I however find often the `Containerfile` format
to be limiting and thus you end up needing to resort to shell scripts anyway.

For more on `bootc` see the [Creating bootc images from scratch](8) in the
Red Hat Enterprise Linux documentation.


## Future
Something else I would like to try is extracting the Alpine kernel from its
`linux-lts` package for direct-boot and to have support for `ext4`. However,
taking it to the next level would be building a kernel image for virtualisation
for use with QEMU, Firecracker and crosvm.

The bigger project is to continue with the idea of taking a container and
running it. That is to drop the need to fetch the container after boot-time
of the VM.

* Alpine + runc + squashfs of the extra OCI image
* Alpine + runc + custom container downloader.
* RedHat-based distribution (bootc) + runc + OCI image

I would like to checkout Compose FS and Torizon OS. The latter
is intended to be fairly minimal and with a container runtime.


## boot2container

This is a bonus section that I looked into before I published this, essentially
I discovered a project which essentially does the kernel + initrd with podman
and configure which container to pull and run via the kernel command line.

The project in question is [boot2container](10).

```sh
curl -OL https://gitlab.freedesktop.org/gfx-ci/boot2container/-/releases/v0.9.17/downloads/initramfs.linux_amd64.cpio.xz
curl -OL https://gitlab.freedesktop.org/gfx-ci/boot2container/-/releases/v0.9.17/downloads/linux-x86_64
curl -OL https://gitlab.freedesktop.org/gfx-ci/boot2container/-/releases/v0.9.17/downloads/linux-x86_64-qemu
```

For Windows, the following runs QEMU using Windows Hypervisor Platform as the
accelerator.
```
qemu-system-x86_64.exe -accel whpx -kernel linux-x86_64-qemu -initrd initramfs.linux_amd64.cpio.xz -nographic -m 384M -append "console=ttyS0 b2c.run=\"-ti docker.io/library/alpine:latest\""
```

### Output
```
[    3.869701] Run /init as init process
2025/10/13 10:11:29 Welcome to u-root!
...[snipped]
[10.38]: About to start executing a container

+ podman start -a 15599e00a88d86d4904aee13f0fd9be72c7c9f5ce2ff1a9e1876d0c40cf72d23
/ # ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
/ # cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.22.2
PRETTY_NAME="Alpine Linux v3.22"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
/ #
```

### boot2container with valkey

In this case the container is valkey, and the port it runs on 6379 is forwarded
to the host.

```
qemu-system-x86_64.exe -accel whpx -kernel linux-x86_64-qemu -initrd initramfs.linux_amd64.cpio.xz -nographic -m 384M -net nic -net user,hostfwd=tcp::16379-:6379 -append "console=ttyS0 b2c.run=\"-ti docker.io/valkey/valkey:8.1.4-alpine3.22""
```

This means from Python can interact with it:
```py
con = valkey.Valkey(host='localhost', port=16379, db=0)
con.set('foo', 'bar')
print(con.get('foo'))
```

A minor improvement to the QEMU command line can be done by adding:
`-nodefaults -serial stdio` which will disable the unused CDROM and video device.

### boot2container verdict
This matches what I had in mind, in terms of initrd with podman. I would be
still keen to try initrd with `runc` or `crun` and the container pre-downloaded
on a secondary boot volume.

Not shown here is it supports adding a disk drive and telling it to
automatically use it as a cache (it will format it as needed).

[0]: {% post_url 2025-05-22-initrd-alpine %}
[1]: https://firecracker-microvm.io/
[2]: https://www.qemu.org/
[3]: https://crosvm.dev/
[4]: https://umo.ci/quick-start/getting-an-image/
[5]: https://bootc-dev.github.io/bootc/intro.html
[6]: https://containers.github.io/bootable/
[7]: https://ostreedev.github.io/ostree/introduction/
[8]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/generating-a-custom-minimal-base-image
[9]: https://github.com/composefs/composefs
[10]: https://gitlab.freedesktop.org/gfx-ci/boot2container


