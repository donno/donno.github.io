---
layout: post
title:  Lima and Windows
date:   2026-05-01 11:00:00 +1030
---

Taking a look at using [Lima](0) which is a is a tool that allows you to run Linux
virtual machines on Windows. The Windows version however uses WSL2 so it
not bring your own kernel compared to if it used Hyper-V and/or the
Windows Hypervisor Platform .

## Set-up

```sh
curl -LO https://github.com/lima-vm/lima/releases/download/v2.1.1/lima-2.1.1-Windows-AMD64.zip
tar xf lima-2.1.1-Windows-AMD64.zip -C D:\Programs\System\lima
```

Trying to start it with `limactl start` failed for two reasons:
* The first was an error converting path.
    * Requires running from Git for Windows (or with its tools on the path).
    * Start `Git Bash` then run the command.
    * See https://github.com/lima-vm/lima/issues/4819
* The second is it didn't have enough configuration even when using
    `limactl start --vm-type=wsl2 --mount-type=wsl2 --containerd=system`.
    * The error this provided was
      ```
      FATA[0000] failed to resolve vm for "C:\\Users\\Donno\\.lima\\default\\lima.yaml": unsupported image type for vmType: wsl2, tarball root file system required: #"https://cloud-images.ubuntu.com/releases/questing/release-20260320/ubuntu-25.10-server-cloudimg-amd64.img"
      ```
    * It failed because it doesn't have a root image preconfigured so it needs
      a YAML file to find the location to the root image, so it expects
       `%USERPROFILE%\.lima\default\lima.yaml`.

## Configuration
In `%USERPROFILE%\.lima\default\lima.yaml`
```yaml
# Example to run Fedora using vmType: wsl2
vmType: wsl2
images:
# Original - this is based on their documentation page as of 2026-04-27
# - location: "https://deps.runfinch.com/common/x86-64/finch-rootfs-production-amd64-1771357941.tar.gz"
- location: "https://deps.runfinch.com/common/x86-64/finch-al2023-rootfs-x86-64-24789639918.tar.gz"
  arch: "x86_64"
  digest: "sha512:1392d0827c1ac5dfed32d01d259f981c859105e7a1d39885ba6db5c2ed5eafbfd26d36a9771f2846c2524daec5b4088858387dda496561fc13aab1ae3a0e1047"
mountType: wsl2
containerd:
  system: true
  user: false
```

**Root file system archive (image)**

Since March 2026, Finch has moved to using Amazon Linux 2023,
`finch-al2023-rootfs-*` rather than what was Fedora 22,
`finch-rootfs-production-*` which in turn was Fedora 20 before it was updated.

The location of which can be found from
[`deps/rootfs.conf`](https://github.com/runfinch/finch-core/blob/24ce9a7ac0622c2f7d97f2429e974d6c965b6332/deps/rootfs.conf)
in the [finch-core repository](https://github.com/runfinch/finch-core/).

## Running Lima

1. Start Lima
    * `lima/bin/limactl.exe start`
    * This will create the WSL2 distribution
2. Run `wsl --list` to confirm there is a `lima-default` distribution
    registered with WSL.
3. Confirm `wsl -d lima-default cat /etc/os-release`
    * I originally used the original root file system  from the documentation
      which was Fedora Linux 42 (Container Image).
    ** With the newer image it will be Amazon Linux 2023.11.20260413.
3. Run  `./lima/bin/limactl.exe shell --start --instance default uname -a`
    * This failed for me. Due to `sshd` failing in the distribution.
    * See below how I got around that.
4. `MSYS_NO_PATHCONV=1 ./lima/bin/limactl.exe shell --start --debug --instance default cat '/etc/os-release'`
    * Setting `MSYS_NO_PATHCONV` is needed when running the command from Git
    Bash (MSYS2) as it has this awkward feature where it decides to expand a
    `/` to `c:/` when passing the argument to an executable.
5. Run container
    * `./limactl shell --preserve-env default nerdctl run -d --name nginx -p 127.0.0.1:8080:80 nginx:alpine`
    * This also failed and I never got to the bottom of it.

## Troubleshooting SSHD
* `limactl shell` failed, no output.
    * Add `--debug` to see what it is doing.
    * The verdict was it runs `ssh` using identify file in
        `%USERPROFILE\.lima\_confi`  to SSH into the machine as `lima`.
        * Seems odd they using `-o User=lima` instead of the `user@host` syntax.
        * Checking via WSL, there is a `/home/lima.guest` directory with ssh setup.
* The issue was sshd isn't running correctly:
    ```
    INFO[0021] [hostagent] Waiting for the essential requirement 1 of 3: "ssh"
    FATA[0601] did not receive an event with the "running" status
    ```
* The problem was the sshd service failed to start in teh container has
    port 22 was already in use. To address I:
    * Installed `nano` via `dnf install nano`
    * Edit Port from 22 to 4022 in `nano /etc/ssh/sshd_config`
    * Restarted the service: `systemctl start sshd.service`
    * Tried sshing to 4022 as `lima` with the identity and it worked.
    * Couldn't however get `limactl` to handle connecting to port 4022 instead of 22.
* End result, found which distribution as causing it, terminated it (`wsl --terminate <ID>`)
    * Changed Port back to 22 in the lima distribution and restarted sshd.
    * Now ssh worked and lima worked.

## Troubleshooting containerd
* `./limactl shell --preserve-env default nerdctl run -d --name nginx -p 127.0.0.1:8080:80 nginx:alpine`
  ```
  FATA[0000] rootless containerd not running? (hint: use `containerd-rootless-setuptool.sh install` to start rootless containerd): stat /run/user/197615/containerd-rootless: no such file or directory
  ```
* This failed, I suspect because the initial `start` failed after ssh which
  re-running start showed that was it, as seen:
  ```
  INFO[0015] [hostagent] Waiting for the optional requirement 2 of 2: "containerd binaries to be installed"
  ```
* `MSYS_NO_PATHCONV=1 ./limactl shell --preserve-env default /usr/local/bin/containerd-rootless-setuptool.sh install`
* This fails with the following output
```
[INFO] Checking RootlessKit functionality
[rootlesskit:parent] error: failed to setup UID/GID map: newuidmap 1370 [0 197615 1 1 524288 1073741824] failed: newuidmap: write to uid_map failed: Operation not permitted
: exit status 1
[ERROR] RootlessKit failed, see the error messages and https://rootlesscontaine.rs/getting-started/common/
```
* `cat /etc/subuid` - shows that is already set-up.
* Both `newgidmap` and `newuidmap` are installed.
* Gave up at this point.
* The command it is trying to run is this:
    `rootlesskit --net=slirp4netns --disable-host-loopback --copy-up=/etc --copy-up=/run --copy-up=/var/lib true`

I ended up stopping there.

## Conclusion

The idea of Lima is nice one if it provides consistent tooling across host
operating systems however if all you need is a solution for Windows it adds a
heavy layer on top of WSL2.

* For containers, you would be better off with Podman, which has Podman Machine
  which can set-up a WSL2 distribution.
* For VMs, you could create your own root file system and install it.
  Essentially, what I explored in [WSL, Alpine and Podman]({% post_url 2022-10-30-osrm %})

After all the current description on the [Lima homepage](0), which as it says
its intending to be similar to WSL2.
> What is Lima?
>
> Lima launches Linux virtual machines with automatic file sharing and port forwarding (similar to WSL2).

[0]: https://lima-vm.io/
