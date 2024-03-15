# Window Containers

This is a dive into using Windows Containers without Docker. The starting point
of this was evaluating the [Host Compute System][0] (HCS) of Microsoft Windows. HCS
provides a setup of APIs for controlling VMs and containers and essentially
built as the low-level API for other container tools such as Docker to be built
upon.

The middle-level system is [hcsshim][1] which is tooling written in Go that
uses HCS and is used in Docker (Moby) and containerd.

I quickly realised that most public code that uses the HCS is written in Go.
The other realisation is rather than trying to use the low-level API it would
be best making sure I could try it out with existing software first to evaluate
what it can do. Enter containerd.

## containerd

* containerd is an container runtime with an emphasis on simplicity,
  robustness, and portability.
* containerd is a graduated member of Cloud Native Computing Foundation (CNCF).
* Supports Windows which is what I was interested in.
* Relatively simple as its made up of a small number of executables.
* Uses plugins to provide additional functionality such as networking.
  * Networking aspect uses Container Network Interface (CNI).

## Downloads
```
curl.exe -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-windows-amd64.tar.gz
curl.exe -LO https://github.com/containerd/nerdctl/releases/download/v1.7.4/nerdctl-1.7.4-windows-amd64.tar.gz
```
The contents of these tarballs are as follows:
* `containerd-1.7.14-windows-amd64.tar.gz`
  * `ctr.exe` - containerd CLI (front-end) - however its unsupported debug and
    administive basically for testing and development. It however provides a
    good way
    to check that the basics work.
  * `containerd.exe` - The container runtime and main program that manages
    the containers.
  * `containerd-stress.exe` - A tool to stress test a containerd daemon.
  * `containerd-shim-runhcs-v1.exe` - I assume this is the tool that containerd
    uses to talk to the Host Compute System on Windows.
* `nerdctl-1.7.4-windows-amd64.tar.gz`
  * `nerdctl.exe` - a command line interface for containerd and is the end-user
    ready tool. The tool aims to be compatible with Docker command line tool.

Future downloads
```
curl.exe -LO https://github.com/microsoft/windows-container-networking/releases/download/v0.3.0/windows-container-networking-cni-amd64-v0.3.0.zip
curl.exe -LO https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-windows-amd64-v1.4.1.tgz
curl.exe -LO https://github.com/moby/buildkit/releases/download/v0.13.0/buildkit-v0.13.0.windows-amd64.tar.gz
```

Ideally a newer version of the first download can be used for networking
however, the [pull request][3] that I used to test this was stuck and not
merged. I since noticed another [pull request][4] that was closed that added
same thing.

The second download might work but requires more set-up to create and configure
the bridge and overlay networks. From what I could tell nerdctl doesn't know
about them so can't help set it up for you.

## Compiling for Network

To allow the containers to contact the Internet (ping, curl, etc), I needed
to build CNI plugin that supported NAT and CNI version 1.0.0. There is no a
prebuilt build.

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

## Things to investigate

* Networking
    * https://github.com/microsoft/windows-container-networking/releases
    - This doesn't seem to work as it doesn't support CNI versions 1.0.0 and only supports 0.2.0 and 0.3.0.

    - https://github.com/containernetworking/plugins/releases is looking like a better approach.
* Building containers (buildx)
  - https://github.com/moby/buildkit/releases/download/v0.13.0/buildkit-v0.13.0.windows-amd64.tar.gz
  * https://github.com/moby/buildkit/blob/master/docs/windows.md
* https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-ContainerdRuntime/install-containerd-runtime.ps1


[0]: https://learn.microsoft.com/en-us/virtualization/api/hcs/overview
[1]: https://github.com/microsoft/hcsshim
[2]: https://github.com/microsoft/windows-container-networking/pull/96
[3]: https://github.com/microsoft/windows-container-networking/pull/101
