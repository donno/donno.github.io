---
layout: post
title:  "Window Containers"
date:   2024-03-16 20:00:00 +0930
---

This is a dive into using Windows Containers without Docker. The starting point
of this was evaluating the [Host Compute System][0] (HCS) of Microsoft Windows. HCS
provides a sets of APIs for controlling VMs and containers and is essentially
built as a low-level API for other container tools such as Docker to be built
upon.

The middle-level system is [hcsshim][1] which is tooling written in Go that
uses HCS and is used in Docker (Moby) and containerd.

I quickly realised that most public code that uses the HCS is written in Go.
The other realisation is rather than trying to use the low-level API straight
away, it would be best making sure I could try it out with existing software
first to evaluate what it can do. Enter containerd.

## containerd

* containerd is a container runtime with an emphasis on simplicity,
  robustness, and portability.
* containerd is a graduated member of Cloud Native Computing Foundation (CNCF).
* Supports Windows which is what I was interested in.
* Relatively simple as its made up of a small number of executables.
* Uses plug-ins to provide additional functionality such as networking.
  * Networking aspect uses Container Network Interface (CNI).

## Downloads
```
curl.exe -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-windows-amd64.tar.gz
curl.exe -LO https://github.com/containerd/nerdctl/releases/download/v1.7.4/nerdctl-1.7.4-windows-amd64.tar.gz
```
The contents of these tarballs are as follows:
* `containerd-1.7.14-windows-amd64.tar.gz`
  * `ctr.exe` - containerd CLI program (front-end) - however it is an
    unsupported debug and administive tool, basically for testing and
    development. It however provides a good way to check that the basics work.
  * `containerd.exe` - The container runtime and main program that manages
    the containers. On Windows, this calls through to the HCS.
  * `containerd-stress.exe` - A tool to stress test a containerd daemon.
  * `containerd-shim-runhcs-v1.exe` - I assume this is the tool that containerd
    uses to talk to the Host Compute System on Windows.
* `nerdctl-1.7.4-windows-amd64.tar.gz`
  * `nerdctl.exe` - a command line interface for containerd and it is the
    end-user ready tool. The tool aims to be compatible with Docker command
    line tool and feature rich.

Future downloads
```
curl.exe -LO https://github.com/microsoft/windows-container-networking/releases/download/v0.3.0/windows-container-networking-cni-amd64-v0.3.0.zip
curl.exe -LO https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-windows-amd64-v1.4.1.tgz
curl.exe -LO https://github.com/moby/buildkit/releases/download/v0.13.0/buildkit-v0.13.0.windows-amd64.tar.gz
```

Ideally a newer version of the first download can be used for networking.
Thatt particular version doesn't work as it doesn't support the newer CNI
format. While the [pull request][3] that I used to test this was stuck and not
merged. I since noticed another [pull request][4] that was closed that added
same thing.

The second download might work but requires more set-up to create and configure
the bridge and overlay networks. From what I could tell `nerdctl` doesn't know
about them so it can't help set it up for you.

## Compiling for Network

To allow the containers to contact the Internet (ping, curl, etc), I needed
to build CNI plugin that supported NAT and CNI version 1.0.0. There was no
prebuilt build at the time of writing.

* To get networking work need the fixes from Paul "TBBle" Hampson for supporting
  CNI 1.0.0 as Microsoft's code only supports 0.2.0 and 0.3.0.
* `git clone https://github.com/TBBle/windows-container-networking.git`
* `git checkout cni_1.0.0-support`
* `make nat`
* Copy the nat.exe to `c:\Program Files\containerd\cni\bin\`

The building was done in Ubuntu within Windows Subsystem for Linux as the
Makefile is set-up to cross-compile for Windows.

You may find that that the master branch from
https://github.com/microsoft/windows-container-networking.git also works as
a different pull request which added 1.0.0 support was merged, Christmas Day
2023.

## Set-up
* Extract the containerd tarball.
* Create the default config
  ```
  bin\containerd.exe config default > config.toml
  ```
* Copy `nat.exe` built from the `windows-container-networking` to
  `C:\Program Files\containerd\cni\bin`.

  The default configuration expects plugins to be in
  `c:\Program Files\containerd\` and you could customise it if needed.

## Running

### Daemon
```
bin\containerd.exe --config config.toml
```

### Client
```
bin\ctr.exe image pull mcr.microsoft.com/windows/nanoserver:10.0.19041.1415
bin\ctr.exe image pull quay.io/quay/busybox:latest

nerdctl.exe pull mcr.microsoft.com/windows/nanoserver:ltsc2019-amd64
nerdctl.exe run --rm --isolation=hyperv --net none mcr.microsoft.com/windows/nanoserver:ltsc2019-amd64

nerdctl.exe run mcr.microsoft.com/windows/nanoserver:2004-amd64 -- cmd /c echo hello
nerdctl.exe run mcr.microsoft.com/windows/nanoserver:2004-amd64 -- ping google.com
```

* The busybox pull failed with error about "failed to extract layer" stating
  the "The program issued a command but the command length is incorrect".
* The nerdctl run with Hyper-V isolation works for nanoserver.
    * Without the -amd64 on the end it fails.
* For my machine the combination of 2004-amd64 works without the isolation mode.

If you want to stick to the basics then the ctr tool is fine but if you are
looking to almost daily drive using containerd then nerdctl is what you want.

### Minimal Working Steps.
```
bin\containerd.exe config default > config.toml
bin\containerd.exe --config config.toml
nerdctl.exe run --rm --isolation=hyperv --net none mcr.microsoft.com/windows/nanoserver:ltsc2019-amd64
```

* As I understand it, the Hyper-V isolation should allow you to run containers
  that don't match your host Windows version.
* The `--net none` option simply means it works without networking so you don't
  need to set that up.

## Building images
The next thing to investigate is building container images.

The `nerdctl.exe build` command issues an error:
> error msg="`buildctl` needs to be installed and `buildkitd` needs to be running, see https://github.com/moby/buildkit"

My timing of going down this whole rabbit hole wasn't too bad because about a
week before I did, a new version of buildkit was released (0.13.0) and with it
comes experimental Windows Containers support. This is specifically now
available with a containerd worker and the Windows release artifacts also
contains the buildkitd.exe binary.

* Download
  ```
  curl.exe -LO https://github.com/moby/buildkit/releases/download/v0.13.0/buildkit-v0.13.0.windows-amd64.tar.gz
  ```
* Extract the tarball
  * The tarball contains `buildkitd.exe` (the daemon/backend) and
    `buildctl.exe` which is the front-end / CLI tool.
* Run the daemon: `.\bin\buildkitd.exe`
* To check it works, try building simple hello world image from the **Example
  Build section** of the project's [Windows documentation][4].
  * Create Dockerfile with:
    ```Dockerfile
    FROM mcr.microsoft.com/windows/nanoserver:ltsc2022
    USER ContainerAdministrator
    COPY hello.txt C:/
    RUN echo "Goodbye!" >> hello.txt
    CMD ["cmd", "/C", "type C:\\hello.txt"]
    ```
    As mentioned below my FROM line ended up using the tag 2004-amd64.
  * Create hello.txt with:
    ```
    Hello from buildkit!
    This message shows that your installation appears to be working correctly.
    ```
  * Build the image.
    ```
    buildctl.exe build `
      --frontend=dockerfile.v0 `
      --local context=. \ `
      --local dockerfile=. `
      --output type=image,name=example.com/buildkit-test/hello-buildkit,push=false
    ```

### Trouble-shooting
* No match for platform in manifest.
  * The full message looks like this
    ```
    error: failed to solve: mcr.microsoft.com/windows/nanoserver:2004: failed to resolve source metadata for mcr.microsoft.com/windows/nanoserver:2004: no match for platform in manifest: not found`
    failed to resolve source metadata for mcr.microsoft.com/windows/nanoserver:2004: no match for platform in manifest: not found
    ```
  * Fix is to modify the file so the tag is `ltsc2022-amd64`.
  * Unconfirmed but the reason seems to be Microsoft published several
    multi-arch images and its confusing the tooling.
* RUN command fails to create the container.
  * The full message looks likes this:
    ```
    error: failed to solve: process "cmd /S /C echo \"Goodbye!\" >> hello.txt" did not complete successfully: failed to create shim task: hcs::CreateComputeSystem tx71wvgg9bgup1xrkkllqlgpc: The container operating system does not match the host operating system.: unknown
    ```
  * The workaround for me was to find the image that matches my host, in my
    case 2002-amd64.
  * Unconfirmed but again the reason seems to be same as above, however in this
    case to run the nanoserver ltsc2022-amd64 image through `nerdctl.exe`, I
    would need to do `--isolation=hyperv` and I haven't been unable to find
    the equivalent setting ot make that the default for containerd or have
    buildkit do the same.

### What's next?

With the **buildkit** support being so fresh the `nerdctl` tool doesn't yet
work with it. The command line help shows that the default address is the pipe
that buildkitd.exe runs on. Based on a pull request that was looking to add
Windows support it sounded like most the work is already in place but it was
simply disabled as it couldn't be tested or used before now.

Once this is in place it will mean `nerdctl` should be able to build images and
with the combination of containerd, nerdctl and buildkit a usable container
environment works.

Investigate Linux Containers on Windows (LCOW) with containerd, this seems to
be an unsupported and abandoned project. A few observerations are:

* There was an [experimental LCOW project][6] using LinuxKit that wrapped up in
  2022.
* The containerd config contains references to windows-lcow. I'm haven't done
  a deep dive but it seems the issue with using it is there is no snapshotter
  to support it.
* The hcsshim project still contains code for LCOW.
  * One of the heighest levels seems to be `createLinuxContainerDocument`.
    calling onto `createLCOWSpec`.
  * The facility to create unitiy VM for running a container in it,
    `uvm.CreateLCOW()` is still referenced if the context os contains "linux"
    otherwise it uses `uvm.CreateWCOW()`.
  * The tests for lcow were removed from hcsshim in [pull #1998][7].
* Moby/Docker [removed their][8] support for LCOW in Docker 23.0 in June 2023.

## Other things of interest

The [install-containerd-runtime.ps1][5] by Microsoft might be interesting to
check over. This is more of an all encompassing script for setting it up as it
includes enabling the Windows Features.

It is worth pointing out that Windows Containers running from containerd work
at the same time as Linux Containers from Podman. This may be the result of
Podman on Windows using the Windows Subsystem for Linux to run the containers
rather than using the Host Compute System.

It would be nice if Podman Desktop supported containerd and could talk to
podman-machine for Linux containers and containerd for Windows container.

[0]: https://learn.microsoft.com/en-us/virtualization/api/hcs/overview
[1]: https://github.com/microsoft/hcsshim
[2]: https://github.com/microsoft/windows-container-networking/pull/96
[3]: https://github.com/microsoft/windows-container-networking/pull/101
[4]: https://github.com/moby/buildkit/blob/master/docs/windows.md
[5]: https://github.com/microsoft/Windows-Containers/raw/Main/helpful_tools/Install-ContainerdRuntime/install-containerd-runtime.ps1
[6]: https://github.com/linuxkit/lcow
[7]: https://github.com/microsoft/hcsshim/pull/1998
[8]: https://github.com/moby/moby/issues/39533#issuecomment-1632061097