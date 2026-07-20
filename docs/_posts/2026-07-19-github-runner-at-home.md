---
layout: post
title:  GitHub Runner at Home
date:   2026-07-19 22:00:00 +1030
---

After debating setting up a complete continuous integration at home, I realised
that GitHub's private repositories can use [self-hosted runners](0) for free.
This provided the simplest option as the code was already hosted with GitHub.

There was [plan](2) in late December 2025 by GitHub to charge $0.002 per-minute
for self-hosted runner which was meant to be a maintenance fee for their
hosting of logs, etc but they backtracked on that postponed it to re-evaluate it.

## First Attempt

Set-up the GitLab Runner in the podman-machine distribution in Windows
Subsystem for Linux (WSL). This was promising until I discovered that the
runner doesn't have native podman support and so it work, I would need to turn
on the Docker-compatible mode where it listens on a socket.

## Second Attempt
Set-up a new Alpine Linux distribution in WSL with `docker` available. I had
looked at using Fedora 44 but ended up trying Alpine instead.

From Windows
```sh
cd K:\Downloads\Linux\v3.24\releases\x86_64
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/x86_64/alpine-minirootfs-3.24.1-x86_64.tar.gz
wsl --import Alpine-Docker D:\vms\wsl\alpine-docker K:\Downloads\Linux\v3.24\releases\x86_64\alpine-minirootfs-3.24.1-x86_64.tar.gz
```

From WSL distribution (Alpine)
```sh
apk add docker openrc openrc-user nano tzdata curl bash
rc-update add docker default
adduser donno
```
The rc-update, sets up Docker daemon to start on boot.

Modify `/etc/init.d/docker` to remove `net` in depend as the networking service
doesn't need to start as networking is configured and works with WSL.

### /etc/wsl.conf
The important part of this change is the boot line so it runs `openrc` as the
init system after WSL2 own init.

Since the GitHub Action Runner doesn't want to run as root, I also added
default user. In hindsight I should have called the account `github` or `ci`
instead of `donno`.

```ini
[boot]
command="openrc default"

[automount]
enabled=false

[interop]
enabled=false
appendWindowsPath=false
# Don't need to launch Windows processes from here.

[user]
default=donno
```

Without the boot command running `/etc/init.d/hostname start` will result in
an error such as this, which also means docker/containerd won't start.
```
 * hostname: failed to acquire lock: Bad file descriptor
```
Rather than:
```
 * WARNING: hostname is already starting
```

### Packages
* `nano` as a text editor for editing scripts.
* `docker` for GitHub runner as it requires socket, it doesn't have Podman support.
* `openrc` for service, the miniroot fs is as a container base so doesn't expect to have a init.
* `tzdata` for time zone database and simply to avoid WSL warning.
* `curl` for downloading files.
* `bash` as the scripts for GitHub Runner require bash.

## GitLab Runner for Alpine

Since Alpine uses musl instead of glibc, the binaries provided through GitHub
Releases don't work on Alpine. The solution was to build from source.

This requires a couple of changes to their codebase to build suitable
binaries.


* In `*.csproj` files replace `linux-x64;` with `linux-x64;linux-musl-x64;`
  which essentially adds the musl runtime to the `RuntimeIdentifiers` element.
* In `dev.sh`, add support for accepting `linux-musl-x64`
* In `src/Misc/externals.sh`, add `linux-musl-x64` as an allowed identifier. for
  the linux-x64 package. 
  
### Auto-start

The following is untested.
`/etc/init.d/gitlab-runner`
```sh
#!/sbin/openrc-run

name="gitlab-runner"
description="GitLab Runner"

command=/home/donno/action-runner/run.sh
command_user=donno

# If you use a different config location, change this:
command_background=true
command_args=""

pidfile="/run/${RC_SVCNAME}.pid"

depend() {
  need docker
  after localmount
}
```

Alternatives to using the run.sh. That said. I think the key feature `run.sh`
has going for it is minimal environment.
* `/bin/bash /home/donno/actions-runner/run-helper.sh`
* `/home/donno/actions-runner/bin/Runner.Listener run`

### Known Issue

* Running actions outside of a container fail as it uses node from
  `externals/node24` rather than `external/node24_alpine`.
  * At the time of writing, I didn't modify my patch to account for that. My
    approach was simply set an image to use.
  * My preference would be to have the externals named as
    `node<versions>_glibc` and `node<versions>_musl` and then symlinking
    a `node<version>_host` rather switching between the alpine vs non-alpine
    folder at runtime.

## Other Tidbits

Logs are written into the `_diag/pages/` directory. This is the output of the
runners themselves, so it shows the output of the `run` directives in the
workflows and also shows what commnads the runner ran, such as how the container
is built.

The configuration is stored in the `.runner` file, with OAuth credentials in
`.credentials.

# Future
* Try to fix the musl vs glib detection for the host so it can run the
  JavaScript actions outside a container.

[0]: https://github.com/actions/runner
[1]: https://github.com/github/roadmap/issues/1197#issuecomment-3675835607
