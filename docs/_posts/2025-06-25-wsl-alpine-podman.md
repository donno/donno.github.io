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
FROM public.ecr.aws/docker/library/alpine:3.22.0
ARG MIRROR=https://dl-cdn.alpinelinux.org/alpine/latest-stable/
ADD --chmod=0755 https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.10/x86_64/apk.static /apk.static
RUN /apk.static --arch x86_64 -X $MIRROR/main -X $MIRROR/community -U --allow-untrusted --root /rootfs --initdb add alpine-base
RUN tar c -z -C /rootfs  --numeric-owner -f /alpine-rootfs.tar.gz .
```

The original plan was to use `public.ecr.aws/docker/library/busybox:unstable-musl`
as the base container however the busybox container lacks the SSL certs.

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

## Wolfi - Root File-system (Update 2025-06-26)
This creates a WSL2 distribution using Wolfi from Chainguard.

This one I simply set-up as a single command call to keep it all self-contained.
Otherwise, I would have considered having a "build.sh" and putting that in
current folder and utilise the mount to have that file in the container and
run it.

```sh
podman run --rm -v .:/host --entrypoint /bin/sh docker.io/chainguard/wolfi-base -c "/usr/bin/apk --arch x86_64 -X https://apk.cgr.dev/chainguard/ -U --allow-untrusted --root /rootfs --initdb add wolfi-base chainguard-keys mount umount && touch /rootfs/etc/fstab && tar c -z -C /rootfs  --numeric-owner -f /host/wolfi.tar.gz ."
wsl --install --name "MyWolfi" --from-file .\wolfi.tar.gz
```

The first issue issue was:
```
wsl: Processing /etc/fstab with mount -a failed.
wsl: Failed to mount C:\, see dmesg for more details.
...
wsl: Failed to translate '<item in %PATH% from Windows>'
... Same as above repeated for each directory in %PATH%.
```
Running `dmesg` it shows:
```
[ 7536.455675] WSL (2 - init(MyWolfi)) ERROR: UtilCreateProcessAndWait:685: /bin/mount failed with 2
[ 7536.456840] WSL (1 - init(MyWolfi)) ERROR: UtilCreateProcessAndWait:707: /bin/mount failed with status 0xff00
```

The problem here is /bin/mount doesn't exist. In Alpine, /bin/mount is linked to /bin/bbsuid.
The solution was add `mount` in addition to `wolfi-base`. While its not required
for starting the distribution, it seemed like a good idea to also include
`umount` to be able to unmount a file-system.

The issue is apk's repositories file is missing, so you can't add any extra
packages (you could manually add the `/etc/apk/repositories` with the contents)..
Initially there was the issue where the signature for it were missing too (so
the `--allow-untrusted` argument needs to be used). That problem was addressed by
adding the `chainguard-keys` package.

An alternative to using the Chainguard distribution would be use
`https://packages.wolfi.dev/os/<arch>`, which if this was used the `wolfi-keys`
package needs to be installed.

## Expectation

I am planning on updating this post if I complete the future work, rather than
doing a part 2.
