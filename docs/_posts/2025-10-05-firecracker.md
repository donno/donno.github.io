---
layout: post
title:  "Trying Firecracker"
date:   2025-10-05 15:00:00 +0930
---

Over the past two days I've been looking at running [Firecracker][0] to run a
Ubuntu based VM from Alpine. The biggest challenge was getting the networking
functional.

[Firecracker][0] is a Virtual Machine Monitor (VMM) by Amazon which started from
From [crosvm][2], a key difference is it doesn't support Windows however its
commercial use is for powering AWS Lambda and Fargate.

## Set-up

### Installation

Install the packages needed
```sh
apk add firecracker iptables ifconfig2 squashfs-tools e2fsprogs curl
```

* The alternative to installing `firecracker` would be to download the release
from [GitHub][2].
* `iptables` is used to manage the kernel firewall and NAT. An alternative to explore
  is `nftables`.
* `ifconfig2` is used to configure TAP tunnel as BusyBox `ip` command doesn't
  support that. I couldn't find an alternative ot this other than compiling
  a sample program for configuring it.
* `squashfs-tools` is for unpacking the root file system image.
* `e2fsprogs` is for creating ext4 file system for the root file system.
* `curl` for issuing commands to the Firecracker socket/API.

### Downloads

x86_64
```sh
wget -O vmlinux https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.13/x86_64/vmlinux-6.1.141
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.13/x86_64/ubuntu-24.04.squashfs
```

Generic - this one would work on `aarch64` and `x86_64`.
```sh
ARCH=$(uname -m)
wget -O vmlinux "https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.13/$ARCH/vmlinux-6.1.141"
wget "https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.13/$ARCH/ubuntu-24.04.squashfs"
```

### Prepare Root File System

This is going to unpack the root file system, add SSH keys. In this case it
pulls the keys from GitHub and it will be my keys so if you following this you
should change `donno` to your GitHub user name or change that line to set-up
the `authorized_keys` with the keys of your choice.

```sh
unsquashfs ubuntu-24.04.squashfs
wget -O squashfs-root/root/.ssh/authorized_keys https://github.com/donno.keys
chown -R root:root squashfs-root
truncate -s 1G ubuntu.rootfs.ext4
mkfs.ext4 -d squashfs-root -F ubuntu.rootfs.ext4
```

### Configuration
The configuration that follows is stored in a file called `ubuntu.config`.
```json
{
  "boot-source": {
    "kernel_image_path": "vmlinux",
    "boot_args": "console=ttyS0 reboot=k panic=1",
    "initrd_path": null
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "partuuid": null,
      "is_root_device": true,
      "cache_type": "Unsafe",
      "is_read_only": false,
      "path_on_host": "ubuntu.rootfs.ext4",
      "io_engine": "Sync",
      "rate_limiter": null,
      "socket": null
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 1024,
    "smt": false,
    "track_dirty_pages": false,
    "huge_pages": "None"
  },
  "cpu-config": null,
  "balloon": null,
  "network-interfaces": [],
  "vsock": null,
  "logger": null,
  "metrics": null,
  "mmds-config": null,
  "entropy": null
}
```

Additionally, check that the `kvm` module is loaded:

```sh
lsmod | grep kvm
kvm_intel             409600  0
kvm                  1392640  1 kvm_intel
irqbypass              12288  1 kvm
```

If not load the module
```sh
modprobe kvm
modprobe kvm_intel
```
The `kvm_intel` is if your CPU is Intel, otherwise there is a `kvm_amd`.

## Running

```sh
rm /tmp/fire.socket && firecracker --config-file ubuntu.config  --api-sock /tmp/fire.socket
```

If successful it will show something like this:
```
...
[  OK  ] Finished fcnet.service.

Ubuntu 24.04.2 LTS ubuntu-fc-uvm ttyS0

ubuntu-fc-uvm login: root (automatic login)

Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.1.141 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
root@ubuntu-fc-uvm:~#
```

Next up networking.

### Shutdown
```sh
curl --unix-socket /tmp/fire.socket -X PUT 'http://localhost/actions' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{ "action_type": "SendCtrlAltDel" }'
```

## Networking

This took a while to get working, in the end the [instructions][5] that worked
out, came from the `crosvm` project. I've left it using the name `crosvm_tap`
for the TAP device.

These commands are run on the host.
```sh
HOST_DEV=$(ip route get 8.8.8.8 | awk -- '{printf $5}')
ip tuntap add mode tap user $USER vnet_hdr crosvm_tap
ip addr add 10.200.0.1/24 dev crosvm_tap
ip link set crosvm_tap up
sysctl net.ipv4.ip_forward=1
HOST_DEV=$(ip route get 8.8.8.8 | awk -- '{printf $5}')
iptables -t nat -A POSTROUTING -o "${HOST_DEV}" -j MASQUERADE
iptables -A FORWARD -i "${HOST_DEV}" -o crosvm_tap -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i crosvm_tap -o "${HOST_DEV}" -j ACCEPT
```

The nice thing about this is the first line which determines what the main
network interface is of the host which for me is `eth0` so that saves needing
to worry about if your system uses `eth0`, `ens4` or `enp0s5`.

If you curious about the `iptables` rules take a look at
[Networking with firecracker][3] by [Hans Pistor][4] which covers it in
details.

Replace this property (key and value),
```
"network-interfaces": [],
```
With
```
  "network-interfaces": [
    {
      "iface_id": "net1",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "crosvm_tap"
    }
  ],
```

Now run the VM again:

```sh
rm /tmp/fire.socket && firecracker --config-file ubuntu.config  --api-sock /tmp/fire.socket
```

Next is setting up the VM.

### Manual

Starting with the manual option is good as that way there are less
moving parts and settings to worry about.

Within the guest:
```sh
ip addr add 10.200.0.2/24 dev eth0
ip link set eth0 up
ip route add default via 10.200.0.1 dev eth0
```

Confirm it worked:
```sh
ping 8.8.8.8
```

To set-up resolving:
```sh
namespace 8.8.8.8 >> /etc/resolv.conf
```
Or, bake it into the root FS
```sh
namespace 8.8.8.8 >> squashfs-root/etc/resolv.conf
mkfs.ext4 -d squashfs-root -F ubuntu.rootfs.ext4
```

8.8.8.8 is Google's DNS server. this will allow `ping bing.com` to work.

If that works, then next is setting it up so we don't need to enter the `ip`
commands after the guest is started.

### Automated

The guest network configuration can be configured using the kernel command
line. At the end of the `boot_args` in the config file we can add:
```
ip=10.200.0.2::10.200.0.1:255.255.255.255::eth0:off"
```

Where the format is `[Guest IP]::[TAP IP]:[Long mask of guest CIDR]::[GuestInterface]`.
The official documentation is found at [Documentation/filesystems/nfs/nfsroot.txt][9]
in the kernel. The feature was originally for NFS but was generalised (renamed
from nfsaddrs to ip).
When I came across this I discovered its possible for the DNS servers to be
provided.

```
"boot_args": "console=ttyS0 reboot=k panic=1 pci=off ip=10.200.0.2::10.200.0.1:255.255.255.255::eth0:off"
```

# Experiment

## Benchmarking

Start a HTTP server:
```sh
apk add uv
uv run -m http.server 15055
```

Before running running firecracker call:
```sh
curl --silent --head http://127.0.0.1:15055/launch > /dev/null

# For example:
rm /tmp/fire.socket && curl --silent --head http://127.0.0.1:15055/launch > /dev/null && firecracker --config-file ubuntu.config  --api-sock /tmp/fire.socket
```

Update the rootfs to include an extra line in the root file system to query the
HTTP server that is running on the host.
```sh
echo "curl --silent --head http://10.200.0.1:15055/bash_init > /dev/null" >> squashfs-root/root/.bashrc
mkfs.ext4 -d squashfs-root -F ubuntu.rootfs.ext4
```

The server output
```
Serving HTTP on 0.0.0.0 port 15055 (http://0.0.0.0:15055/) ...
127.0.0.1 - - [05/Oct/2025 17:29:17] code 404, message File not found
127.0.0.1 - - [05/Oct/2025 17:29:17] "HEAD /launch HTTP/1.1" 404 -
10.200.0.2 - - [05/Oct/2025 17:29:18] code 404, message File not found
10.200.0.2 - - [05/Oct/2025 17:29:18] "HEAD /bash_init HTTP/1.1" 404 -
```

With minimal implementation of server to change how the time is printed, we
can get the nanoseconds. An implementation which handled the `/launch` and
`/bash_init` requests and computed the time since launch was considered but I
didn't intend to collect multiple samples.

```python
from http.server import HTTPServer, SimpleHTTPRequestHandler
import time

class Handler(SimpleHTTPRequestHandler):
    def log_date_time_string(self):
      return str(time.monotonic_ns())

if __name__ == "__main__":
    httpd = HTTPServer(('', 15055), Handler)
    httpd.serve_forever()
```
With the revisited output.
```
127.0.0.1 - - [30898397784530] "HEAD /launch HTTP/1.1" 404 -
10.200.0.2 - - [30899655117847] "HEAD /bash_init HTTP/1.1" 404 -
127.0.0.1 - - [31139610356885] "HEAD /launch HTTP/1.1" 404 -
10.200.0.2 - - [31140904802901] "HEAD /bash_init HTTP/1.1" 404
```

So with those two it is 1294.446ms and 1267.333ms across two runs.

## Alpine
```sh
wget https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/netboot-3.22.1/initramfs-virt
rm /tmp/fire-alpine.socket && firecracker --config-file alpine.config  --api-sock /tmp/fire-alpine.socket
```

This doesn't use the kernel from Alpine due to the fact it is compressed kernel
and Firecracker needs the uncompressed kernel.

* The [extract-vmlinux][7] script from the Linux kernel repository does not
  work with [BusyBox][8].
  * It needs GNU grep, which is easy to install via `apk add grep`
  * Its use of `mktemp` is different as it uses XXX where BusyBox requires XXXXXX.
    This wouldn't be too difficult to provide a `sed` line to tweak.
  * Some other unknown issue.

The network failed to get set-up automatically despite using hte same tun and
ip kernel configuration as last time. I switched to using the `initrdfs.wnet`
from the post I did for creating a `initrd` for Alpine. The network didn't
work out-of-box, I had to fallback to the manual commands to assign the IP etc
and it was able to ping 8.8.8.8.

The Alpine virtual machine is unable to resolve domains via DNS, which I don't
believe I had a problem with when using QEMU.

Running the following is fine, so there is some issue with the configuration.
```sh
nslookup google.com 8.8.8.8
```

# Future
* Sort out how to set-up multiple guests with their own IP. For this the
"Advanced: Multiple guests" section of the Firecracker [network setup][6] guide
looks like a good starting point.
* Sort out multiple guests in general
* Get Alpine working as the guest OS - with networking pre-configured.
* Try using `nftables` over `iptables`.
* Look into the Firecracker [Jailer][10].

[0]: http://firecracker-microvm.io/
[1]: https://github.com/google/crosvm
[2]: https://github.com/firecracker-microvm/firecracker
[3]: https://hans-pistor.tech/posts/networking-with-firecracker/
[4]: https://hans-pistor.tech/
[5]: https://crosvm.dev/book/devices/net.html
[6]: https://github.com/firecracker-microvm/firecracker/blob/main/docs/network-setup.md#on-the-host
[7]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/scripts/extract-vmlinux
[8]: https://busybox.net/
[9]: https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt
[10]: https://github.com/firecracker-microvm/firecracker/blob/main/docs/jailer.md