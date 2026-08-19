---
---
# Setup
apk add docker add openrc openrc-user nano

## WSL

```sh
cd K:\Downloads\Linux\v3.24\releases\x86_64
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/x86_64/alpine-minirootfs-3.24.1-x86_64.tar.gz
wsl --import Alpine-Docker D:\vms\wsl\alpine-docker K:\Downloads\Linux\v3.24\releases\x86_64\alpine-minirootfs-3.24.1-x86_64.tar.gz
```

## Alpine

```sh
apk add docker openrc openrc-user nano tzdata curl bash
rc-update add docker default
adduser donno
```

Modify `/etc/init.d/docker` to remove `net` in depend as the networking service doesn't need to start as it already works with WSL.

Save the following to `/etc/wsl/conf` then restart distro.
```
[boot]
command="openrc default"

[user]
default=donno
```

The rc-update, sets up Docker daemon to start on boot.

### Packages
* `nano` as a text editor for editing scripts.
* `docker` for GitHub runner as it requires socket, it doesn't have Podman support.
* `openrc` for service, the miniroot fs is as a container base so doesn't expect to have a init.
* `tzdata` for time zone database and simply to avoid WSL warning.
* `curl` for downloading files.
* `bash` as the scripts for GitHub Runner require bash.

## GitHub Runner

```sh
mkdir actions-runner &&
   curl -o actions-runner/actions-runner-linux-x64-2.335.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.335.1/actions-runner-linux-x64-2.335.1.tar.gz
tar xzf ./actions-runner/actions-runner-linux-x64-2.335.1.tar.gz --directory=actions-runner
```

# Troubleshooting

```sh
/etc/init.d/hostname start
```

Resulted in
```
 * hostname: failed to acquire lock: Bad file descriptor
```
In another distribution it was:
```
 * WARNING: hostname is already starting
```

Check
```
openat(-1, "exclusive", O_RDONLY|O_LARGEFILE|O_CLOEXEC|O_DIRECTORY) = -1 EBADF (Bad file descriptor)
# Verses
openat(4, "exclusive", O_RDONLY|O_LARGEFILE|O_CLOEXEC|O_DIRECTORY) = 14
```

Fix configure `/etc/wsl.conf` boot option to cause openrc to start when WSL's init finishes.



