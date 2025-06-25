---
layout: post
title:  "WSL, Alpine and Podman"
date:   2025-06-25 23:25:00 +0930
---

After an experiment with running Podman within an Alpine distrubution in WSL2
was successfully, I wondered could I make a minimal distrubution that had
Podman pre-installed ready to go.

This distribution is intended to be an alternative to the Fedora-based WSL
(Windows Subsystem for Linux) distribution that Podman Desktop set-up for you.

This only covers the creation of the WSL2 distribution and not the original
set-up of setting-up `podman`.

## Starting point
Creating a the root file-system tarball that `wsl.exe` can use to install the
distribution.

This is simply to check the basics work.

### Containerfile.alpineroot
```Containerfile


#FROM public.ecr.aws/docker/library/busybox:unstable-musl
# Can't use busybox container as it lacks the SSL certs.

FROM public.ecr.aws/docker/library/alpine:3.22.0
ARG MIRROR=https://dl-cdn.alpinelinux.org/alpine/latest-stable/
ADD --chmod=0755 https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.10/x86_64/apk.static /apk.static
RUN /apk.static --arch x86_64 -X $MIRROR/main -X $MIRROR/community -U --allow-untrusted --root /rootfs --initdb add alpine-base
RUN tar c -z -C /rootfs  --numeric-owner -f /alpine-rootfs.tar.gz .
```

### Usage
```
podman build -f Containerfile.alpineroot  --tag alpineroot:rootfs
podman run --rm -it -v .:/host localhost/alpineroot:rootfs cp /alpine-rootfs.tar.gz /host
wsl --install --name "MyAlpine" --from-file .\alpine-rootfs.tar.gz
```

1. Build the container image
2. Run the container image and copy the resulting root file-system tarball out
3. Install the root file-system as a WSL distribution.

Ideally, this would remove the container image (i.e. `rmi`)

### Result

* The compressed tarball was 4.4MB.
* Within WSL2, this resulted in a 11.4 root partition.
* Alpine functioned as normal.

For comparison you can use the existing root file-system provided.
```
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.0-x86_64.tar.gz
wsl --install Alpine322 --fixed-vhd --name "Alpine3.22" --from-file alpine-minirootfs-3.22.0-x86_64.tar.gz
```

## Podman

### Containerfile.alpinepod
```
FROM public.ecr.aws/docker/library/alpine:3.22.0
ARG MIRROR=https://dl-cdn.alpinelinux.org/alpine/latest-stable/
ADD --chmod=0755 https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.10/x86_64/apk.static /apk.static
RUN /apk.static --arch x86_64 --no-cache -X $MIRROR/main -X $MIRROR/community -U --allow-untrusted --root /rootfs --initdb add alpine-baselayout-data busybox podman ca-certificates iptables
RUN mkdir /rootfs/var/tmp
# Alternative to the above is to install alpine-baselayout (which also installs alpine-baselayout-data)
RUN chroot /rootfs /usr/sbin/update-ca-certificates
RUN tar c -z -C /rootfs  --numeric-owner -f /alpine-pm-rootfs.tar.gz .
```

* `iptables`is needed for `netavark` used by podman.
* `ca-certificates` and the call to `update-ca-certificates` is needed for the
  SSL certificates for `podman` to be able to fetch from container registries
  requested via HTTPS.
* Creation of /var/tmp is to fix the error:
  ```
  Error: internal error: unable to copy from source docker://quay.io/podman/hello:latest: initializing destination containers-storage:[overlay@/var/lib/containers/storage#+/run/containers/storage:overlay.mountopt=nodev]quay.io/podman/hello:latest: creating a temporary directory: stat /var/tmp: no such file or directory
  ```
* `--no-cache` prevents the APK indices being kept in /var/cache/apk/ in the
  resulting root file-system.

### Usage
```
podman build -f Containerfile.alpinepod  --tag alpinepod:rootfs
podman run --rm -it -v .:/host localhost/alpinepod:rootfs cp /alpine-pm-rootfs.tar.gz /host
wsl --install --name "MyAlpine" --from-file .\alpine-rootfs.tar.gz
```

1. Build the container image
2. Run the container image and copy the resulting root file-system tarball out
3. Install the root file-system as a WSL distribution.

Ideally, this would remove the container image (i.e. `rmi`)

### Problems

**WSL Init**

This problem was not critical as the distribution still booted and you can then
run `podman`, however it was bugging me.
```
<3>WSL (12 - Relay) ERROR: CreateProcessParseCommon:1008: getpwuid(0) failed 2
wsl: Processing /etc/fstab with mount -a failed.
<3>WSL (12 - Relay) ERROR: operator():519: getpwuid(0) failed
```

This was fixed by installing `alpine-baselayout-data`. Not sure if this is the
`/etc/fstab` or also needs `/etc/shadow`, `/etc/passwd` for user (see below).

**Unknown uid**

For example running `whoami` reports unknown uid.
```
~ # whoami
whoami: unknown uid 0
```

This is fixed by `alpine-baselayout-data` so best to simply install that.
It is less than 20KiB.

### Result

The following output (and input echo) includes output from the WSL set-up
command which by default installs and then runs it.
```
Installing: .\alpine-pm-rootfs.tar.gz
Distribution successfully installed. It can be launched via 'wsl.exe -d AlpineTest'
Launching AlpineTest...

~ # podman run --rm -it quay.io/podman/hello
Trying to pull quay.io/podman/hello:latest...
Getting image source signatures
Copying blob 81df7ff16254 done   |
Copying config 5dd467fce5 done   |
Writing manifest to image destination
!... Hello Podman World ...!

[rest of output excluded]
```

* The compressed tarball was 35MB
* The uncompresed tarball was 95MB
* Within WSL2, this resulted in a 93MB root partition.
* Alpine functioned as normal.

### Future Work
* Possibly set-up it up to use rootless
* Set-up ssh server and socket to allow `podman` running as a native Windows
  process to connect and use it.

## Tid-bits
* Tried using `busybox:unstable-musl` as the base image but it lacks the SSL
  certificates for apk.static to be able to resolve the package registry.
* To install extra packages after installing the disturubtion, the `apk.static`
  can be downloaded and then extra packages added i.e.
  ```sh
  wget https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.10/x86_64/apk.static
  chmod +x ~/apk.static
  ./apk.static  --allow-untrusted -X https://dl-cdn.alpinelinux.org/alpine/latest-stable/main add nano
  ```

## Expectation

I am planning on updating this post if I complete the future work, rather than
doing a part 2.
