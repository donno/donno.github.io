---
---
The following was run in an Alpine container.

```sh
apk add abuild alpine-conf syslinux xorriso squashfs-tools grub mtools doas git
adduser builder
su -l builder
git clone --depth=1 https://gitlab.alpinelinux.org/alpine/aports.git
abuild-keygen -a -i
# Disclaimer: THe follow needs to be based on what is done above.
export PACKAGER_PRIVKEY=/home/builder/.abuild/builder-69f74891.rsa
./aports/scripts/mkimage.sh --repository http://172.27.176.1/mirror/alpine/v3.22/main/
```

* This is using the local mirror on the host machine at:
    `http:// 172.27.176.1/mirror/alpine/v3.22/`
* It may be possible to have `git clone` do even less by focusing only on
  the script directory.
* Need to add `--profile` so it only builds one image instead of all.

## Output
```
af8a53c162ec:~$ ./aports/scripts/mkimage.sh --repository http://172.27.176.1/mirror/alpine/v3.22/main/
OK: 0 B in 0 packages
v3.22.3-105-g58283386ebd [http://172.27.176.1/mirror/alpine/v3.22/main]
OK: 5646 distinct packages available
>>> mkimage-x86_64: Building minirootfs
>>> mkimage-x86_64: Creating alpine-minirootfs-260503-x86_64.tar.gz
>>> mkimage-x86_64: Building netboot
>>> mkimage-x86_64: --> kernel x86_64 lts ac0f25db8eb54040aaf546701673f0b48610d416 linux-lts linux-firmware wireless-regdb
Parallel mksquashfs: Using 8 processors
Creating 4.0 filesystem on /tmp/update-kernel.mlNcfm/boot/modloop-lts, block size 131072.
...
By defualt this creates everything.
```

## Troubleshooting
* `ERROR: --usermode not allowed as root`
    * Fix: Run the command under teh `nobody` user by prepending the command
     with `su -s /bin/sh nobody`.
        * Setting the shell to `/bin/sh` with the `-s` argument is because it
          is configured with no login shell by default and would output 
          > This account is not available
    * Reference: https://coders-home.de/en/alpine-3-23-mkimage-sh-error-usermode-not-allowed-as-root-1720.html
* `ERROR: Need $PACKAGER_PRIVKEY to be set for modloop_sign=yes`
    * Fix: Run `abuild-keygen -a -i`


## Failed Ideas
* Using the `nobody` user
  ```sh
  mkdir /.abuild && chown nobody:nobody /.abuild && su -s /bin/sh nobody -c "abuild-keygen -a -i"
  su -s /bin/sh nobody -c "./scripts/mkimage.sh --repository http://172.27.176.1/mirror/alpine/v3.22/main/"
  ```
    * This likely would have worke.d*

echo "PACKAGER_PRIVKEY=\"$privkey\"" >> "$ABUILD_USERCONF"


## Other Notes

* `apk index` -> create repository index file from packages
    * A use case for this would be booting Alpine from a FAT-32 USB and
      including additional packages so they don't need to be fetched from
      the Internet.

## TODO

* Create profile with Gnome installed.
* Look at `genapkovl-mkimgoverlay.sh`
  * Example: https://gitlab.alpinelinux.org/alpine/aports/-/blob/master/scripts/genapkovl-dhcp.sh?ref_type=heads

