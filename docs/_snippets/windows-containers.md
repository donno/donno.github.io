# Window Containers

The `cri-containerd-(cni-)` packages are deprecated and will be removed in
containerd 2.0, however they do have a cni folder.

## Downloads
```
curl.exe -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-windows-amd64.tar.gz
curl.exe -LO https://github.com/containerd/nerdctl/releases/download/v1.7.4/nerdctl-1.7.4-windows-amd64.tar.gz
```

Future downloads
```
curl.exe -LO https://github.com/microsoft/windows-container-networking/releases/download/v0.3.0/windows-container-networking-cni-amd64-v0.3.0.zip
curl.exe -LO https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-windows-amd64-v1.4.1.tgz
```

* Ideally a newer version of the first download can be used for networking however the merge request is stuck.
* The second download might work but requires more set-up to create/config the bridge and overlay networks.

## Set-up
The default configuration expects plugins to be in `c:\Program Files\containerd\`

* Create the default conifg
  ```
  bin\containerd.exe config default > config.toml
  ```

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
```

* The busybox pull failed with error about "failed to extract layer" stating
  the "The program issued a command but the command length is incorrect".
- The nerdctl run with Hyper-V isolation works for nanoserver.
    * Without the -amd64 on the end it fails.


### Minimal Working steps.

```
bin\containerd.exe config default > config.toml
bin\containerd.exe --config config.toml
nerdctl.exe run --rm --isolation=hyperv --net none mcr.microsoft.com/windows/nanoserver:ltsc2019-amd64
```

## Things to investigate

* https://github.com/containerd/nerdctl - Already looks promising way to work with it.
* Networking -
    https://github.com/microsoft/windows-container-networking/releases 
    
    - This doesn't seem to work as it doesn't support CNI versions 1.0.0 and only supports 0.2.0 and 0.3.0.
    - Support for CNI 1.0.0 stalled - https://github.com/microsoft/windows-container-networking/pull/96
    - https://github.com/containernetworking/plugins/releases is looking like a better approach.
* https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-ContainerdRuntime/install-containerd-runtime.ps1