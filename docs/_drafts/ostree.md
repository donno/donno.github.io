

TODO: Write a introduction.

There are two ways to go when looking at `ostree`, using the command line
interface (CLI) or the more application programming interface (API). Starting
with the CLI version is simpler and less overhead to get started (as you 
don't need to start writing code) and it what the introduction shows.
The downside is it harder ot keep track of what you did unless you write it
as a shell script.

## Getting Started

Starting out on Alpine Linux, the first step is to install the `ostree` tooling
by installing the package with it.

```sh
apk add ostree
```

Create a new OSTree repository in a folder called `alpine_repo` using the
sub-command `init`.

```sh
ostree --repo=alpine_repo init
```

Prepare a tree (directory) to add and add mini root file-system for Alpine as
the starting point for a more detailed example. The official introduction 
starts with creating a text file and adds that instead.

Download the mini root filesystem for Alpine.
```sh
wget https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.2-x86_64.tar.gz
```

Commit the contents of the root file system as the first commit in a new branch.
```sh
ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --tree=tar=alpine-minirootfs-3.22.2-x86_64.tar.gz
 --subject "Add minimal root file system for Alpine 3.22.1 (x86_64)"
```
As implied by the above command, the ostree tooling supports operating directly
from a tar file.

The alternative would be to:
```sh
([ -d starting-tree ] || mkdir starting-tree) && tar xf alpine-minirootfs-3.22.2-x86_64.tar.gz -C starting-tree
ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --tree=dir=starting-tree
 --subject "Add minimal root file system for Alpine 3.22.1 (x86_64)"
```

Can query the reference which should list the branch we just made
```sh
ostree --repo=alpine_repo refs
```
Outputs: `alpine-v3.22.2-x86_64`

Check out the contents
```sh
ostree --repo=alpine_repo ls alpine-v3.22.2-x86_64
```
Check out the contents of a file
```sh
ostree --repo=alpine_repo cat alpine-v3.22.2-x86_64 /etc/os-release
```

To checkout the entire tree:
```sh
ostree --repo=alpine_repo checkout alpine-v3.22.2-x86_64 tree-checkout
```

## Installing packages

On the [Writing a buildsystem and managing repositories][1] page of ostree's
documentation, it mentions there are are systems the following the
pattern:
```sh
$pkg --installroot=/path/to/tmpdir install foo bar baz
$imagesystem commit --root=/path/to/tmpdir
```

Where `$pkg` would be `apk` in the case of Alpine Linux, but otherwise 
would be `yum` or `apt-get`.

So with that in mind lets install `curl` in a new commit.

### Installing curl
If you are following along on a system other than Alpine, use apk.static.

The aspect that is unclear would the process be to checkout the latest commit,
run `apk add` in that checkout, then commit and thus let it handle the diff.
Or do you install to a new directory and then commit it and if so how do you
tell it what is already installed? A way that comes to mind for the second
approach is to mount a overlayfs with the checkout and a working/writable area
which would be the directory to commit afterwards.

### First Option
This is the option where a checkout is down of the ostree adn then it is 
isntalled there.

```sh
ostree --repo=alpine_repo checkout alpine-v3.22.2-x86_64 staging-area
apk add --root staging-area --no-cache curl
ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --tree=dir=staging-area --subject "Adding curl."
```

Can see the changes with:
```sh
ostree diff --repo=alpine_repo alpine-v3.22.2-x86_64
```
In this case, the output is:
```
M    /etc/apk/world
M    /lib/apk/db/installed
M    /lib/apk/db/scripts.tar
A    /usr/bin/curl
A    /usr/bin/wcurl
A    /usr/lib/libbrotlicommon.so.1
A    /usr/lib/libbrotlicommon.so.1.1.0
A    /usr/lib/libbrotlidec.so.1
A    /usr/lib/libbrotlidec.so.1.1.0
A    /usr/lib/libbrotlienc.so.1
A    /usr/lib/libbrotlienc.so.1.1.0
A    /usr/lib/libcares.so.2
A    /usr/lib/libcares.so.2.19.4
A    /usr/lib/libcurl.so.4
A    /usr/lib/libcurl.so.4.8.0
A    /usr/lib/libidn2.so.0
A    /usr/lib/libidn2.so.0.4.0
A    /usr/lib/libnghttp2.so.14
A    /usr/lib/libnghttp2.so.14.28.4
A    /usr/lib/libpsl.so.5
A    /usr/lib/libpsl.so.5.3.5
A    /usr/lib/libunistring.so.5
A    /usr/lib/libunistring.so.5.2.0
A    /usr/lib/libzstd.so.1
A    /usr/lib/libzstd.so.1.5.7
A    /var/cache/apk/APKINDEX.6c16d705.tar.gz
A    /var/cache/apk/APKINDEX.76ae5dea.tar.gz
```

Can test it:
```sh
ostree --repo=repo checkout alpine-v3.22.2-x86_64 with-curl
chroot with-curl
curl --help
```

With those commands curl won't work as the no way to resolve the host within
the new root, as changing the root and having working network requires extra
steps.

The idea of construction trees with unions - where packages are committed to
separate branches is an interesting approach.

## Next
This can't be turned into a bootable image as there is no kernel.

Nominally would need to install `linux-virt`, similar to how curl was installed.

```sh
apk add --root staging-area  --no-cache linux-virt
```

This is where I initially stopped for the day. This was originally done around
2025-10-15.

The other thing to consider is `rpm-ostree` , which it takes as input RPMs, and
commits them. This means there could be a `apk-ostree` created which would
take an input APK (Alpine Package Keeper file) and commit them too.

[1]: https://ostreedev.github.io/ostree/buildsystem-and-repos/
