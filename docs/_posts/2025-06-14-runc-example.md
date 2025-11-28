---
layout: post
title:  "Examples of runc"
date:   2025-06-14 18:00:00 +0930
---

A brief demonstration of using [`runc`][0] directly to create a container.
`runc` is a low-level tool meant to be used by higher-level tooling like
Docker and Podman. This example is from their README but uses `apk`
from Alpine to create a root file-system instead of extracting one from
an existing container with `docker`.

## Set-up

### Host

For Alpine Linux, install the `runc` package, via `apk add runc` or if
you you want to try `crun` instead then `apk add crun`.

The two packages are:
* `runc` is CLI tool for spawning and running containers according to the OCI specification and is implemented in Go.
* `crun` is a Fast and lightweight fully featured OCI runtime and C library for running containers

With the key difference being `runc` is implemented in Go where as `crun` is written in C.

### OCI Bundle
Create the root file-system and the spec (JSON file).
```sh
curl -LO https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.6/x86_64/apk.static
mkdir /opt/busybox-container
./apk.static --arch x86_64 -X https://dl-cdn.alpinelinux.org/alpine/edge/main/ --root /opt/busybox-container/rootfs --initdb --no-cache --allow-untrusted add busybox
(cd /opt/busybox-container && runc spec)
```

## Run it
The `runc start` command looks in the current working directory for the `config.json` file, created by `runc spec`.

```sh
cd /opt/busybox-container
runc run busybox-1
```

You can see it running with `runc list`, for example on my system
```sh
~# runc list
ID          PID         STATUS      BUNDLE                   CREATED                          OWNER
busybox-1   188         running     /opt/busybox-container   2025-06-14T04:22:40.170715447Z   root
```

The same bundle can be run with `crun`.

```sh
cd /opt/busybox-container
crun run bb-1
```

Likewise you can see it running, which is picked up differently from `runc`.
```sh
crun list
NAME PID       STATUS   BUNDLE PATH                             CREATED                        OWNER
bb-1 266       running  /opt/busybox-container                  2025-06-14T04:35:18.265240Z    root
```

## Filesystem writing with overlay FS
This section was added 2025-07-05.

If you simply, change `root.readonly` to true, it means any changes when running
will be written back to the `rootfs` so you would need to make a new `rootfs`
each time you want a fresh copy.

For this we will create an overlay filesystem similar to how higher-level
container runtimes deal with it.

There are four parts to mounting an overlay file system.
- Lower directory - this is read-only access and is the base layer. In this case
  this will be the `rootfs` directory created above.
- Upper directory - this is where modifications will be stored. This will be a
  new directory, lets call it `goofy_davinci`.
- Working directory - where work in progress is written before being written to
  the upper directory, this will be `goofy_davinci-work`.
- Mount point - the directory where the filesystem will be mounted.

In this case you can think of the lower directory being a single-layer
container image, the upper directory is the file system for the particular
container instance.

```sh
mkdir /opt/busybox-container/goofy_davinci /opt/busybox-container/goofy_davinci-work /opt/busybox-container/goofy_davinci-merged
mount -t overlay overlay -o lowerdir=/opt/busybox-container/rootfs,upperdir=/opt/busybox-container/goofy_davinci,workdir=/opt/busybox-container/goofy_davinci-work /opt/busybox-container/goofy_davinci-merged
```

So if you wanted to take the same base image and run another instance you could do:
```sh
mkdir  /opt/busybox-container/gifted_jepsen /opt/busybox-container/gifted_jepsen-work
mount -t overlay overlay -o lowerdir=/opt/busybox-container/rootfs,upperdir=/opt/busybox-container/gifted_jepsen,workdir=/opt/busybox-container/gifted_jepsen-work /opt/busybox-container/gifted_jepsen-merged
```

Open the `conifg.json` and change `root.path` to `goofy_davinci-merged` and
`root.readonly` from `true` to `false`.

Now lets run it with:
```sh
runc run goofy_davinci
```

Within the container create some folders / files:
```sh
mkdir -p /hello/world
date >> /hello/world/today
exit
```

Now on the host, you should have
`cat /opt/busybox-container/goofy_davinci/hello/world/today` which will output
the current date and time.

The alternative to this is to use mount points so you can add content from the
host and then do it.

## File system writing with mounts
This section was added 2025-07-05.

This set-up gives you a path which you can write to within the container,
similar to the `--volume` argument on high level container engines such as
Podman and Docker.

### Set-up
Create the directory where we will be mounting.
Populating it with some files for later.
```sh
mkdir /root/from_container
echo "root says hello" >> /root/from_container/hello
```

Next modifiy the `config.json` to add the mount point, within the `mounts`
array, add the following item and remmeber to add comma into the array as
needed.

```json
{
        "destination": "/host",
        "type": "none",
        "source": "/root/from_container",
        "options": ["rbind","rw"]
}
```

This will mount the `"/root/from_container` directory within the container as
`/host` with read-write access.

### Run

```sh
crun run bb-1
```

Within the container run:
```sh
# cat /host/hello
root says hello
# echo "container says bye" >> /host/bye
# exit
```
Now outside the contain if you `cat /root/from_container/bye` it should say
`container says bye`.

## Windows

I came across `runhcs` which as Microsoft mentions on their
[container platform tools page][2], `runhcs` is the Windows container host
counterpart to runc.

A quick heads-up, I don't get it working.

It a fork of `runc` for running applications packaging according to OCI format
however it runs natively on Microsoft Windows and can run either Windows or
Linux Hyper-V isolated containers. It can also run Window process containers, 
which are commonly called process isolation in Microsoft's documentation.

I was curious about the Linux containers as that sounds like it without needing
to build upon WSL2 and presumably works by starting a Linux VM.

I built `runhcs.exe`, `uxboot.exe` and `gcs` which is the Guest Container
Services program for Linux.

The bundle from above was copied over to Windows to try to run.
```sh
runhcs.exe run bb-1 --shim-log shim.log --vm-log vm.log 
```

This fails because it has no host / utility VM to use, so I looked at setting
up one. I come across `uvmboot`

```sh
uvmboot.exe -gcs lcow --boot-files-path D:\containers --kernel-file vmlinux --root-fs-type vhd -t exec "/bin/sh"
```

For the kernel, I grabbed the Alpine kernel image that I used with CrosVM and
run the `extract-vmlinux` script to extract the uncompressed vmlinux file from
it (i.e. a `vmlinuz` file).

From the the [hcsim][4] repository, I followed the instructions to build the
Linux guest agent as well as a rootfs.
```sh
# Start by grabbing base.tar.gz from busybox - I don't think this is what is needed.
podman run -it docker.io/library/busybox:latest
# In second terminal run
podman export <container-id> --output base.tar
gzip base.tar
make -f Makefile.bootfiles out/rootfs.vhd
```
Copy the resulting `rootfs.vhd` to the boot files path mentioned above.

That makefile can also build a `out/kernel.bin` target however it requires
a IGVM tool which I didn't follow-up on.

This however didn't work, eventually it would start and then timeout.
The `hcsdiag list` command would show that the VM was running.
```
time="2025-06-14T20:28:54+09:30" level=error msg="failed to connect to entropy socket" error="context deadline exceeded" uvm-id=uvmboot-0
time="2025-06-14T20:28:54+09:30" level=error msg="failed to connect to log socket" error="context deadline exceeded" uvm-id=uvmboot-0
time="2025-06-14T20:28:54+09:30" level=error msg="failed to run UVM" error="failed to connect to entropy socket: context deadline exceeded" uvm-id=uvmboot-0
```

Even trying to run the container while it says running it would still fail.
```sh
runhcs.exe run bb-1 --shim-log shim.log --vm-log vm.log --host uvmboot-0
```

Looking at the [Test-LCOW-UVM.ps1][5] script to see how they test it, I tried
the more complete command line they use:
```sh
uvmboot.exe -gcs lcow -fwd-stdout -fwd-stderr -output-handling stdout -boot-files-path d:\containers -root-fs-type vhd -kernel-file vmlinux-mount-scsi d:\containers\rootfs.vhd --tty --exec ash
```

Unfortunately, the artifacts used in testing are pulled from Azure and are not
public. This would include `kernel`, `vmlinux` and a rootfs tarball. The GitHub
Action for the hcsshim project get the boot files by essentially doing this:
```sh
az extension add --name azure-devops
az artifacts universal download --organization "https://msazure.visualstudio.com/" --project "ContainerPlatform" --scope project --feed "ContainerPlat-Dev" --name "azurelinux-uvm" --version "*.*.*" --path ./downloaded_artifacts
```

I didn't end-up trying a Windows container bundle.

[0]: https://github.com/opencontainers/runc
[1]: https://github.com/containers/crun
[2]: https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd
[3]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/extract-vmlinux
[4]: https://github.com/microsoft/hcsshim

