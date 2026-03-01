
My dive into ostree, which is system of versioning updates for Linux-based
operating systems.

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
ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --tree=tar=alpine-minirootfs-3.22.2-x86_64.tar.gz \
 --subject "Add minimal root file system for Alpine 3.22.1 (x86_64)"
```
As implied by the above command, the ostree tooling supports operating directly
from a tar file.

The alternative would be to:
```sh
([ -d starting-tree ] || mkdir starting-tree) && tar xf alpine-minirootfs-3.22.2-x86_64.tar.gz -C starting-tree
ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --tree=dir=starting-tree \
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
ostree --repo=alpine_repo cat alpine-v3.22.2-x86_64 /usr/lib/os-release
```

**Gotcha**: The path `/etc/os/release` doesn't work as it `ostree` seems to
struggle with the symlinks.


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
This is the option where a checkout is performed from ostree and then it is
install there.

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

### Second Option

Follow the pattern from the documentation and utilise the `--installroot`
command.

Writing a script to help with this would be ideal where it would configure
`apk` based on the contents within `ostree`, but since this is the first time
need to figure out the steps anyway.

1. Determine repository paths
    ```sh
    ostree --repo=alpine_repo cat alpine-v3.22.2-x86_64 /etc/apk/repositories > tmp.repositories
    ```
2. Install the packages into a folder
    ```sh
    apk add --initdb --allow-untrusted --logfile=no --repositories-file tmp.repositories --root pkg.curl --no-cache curl
    ```
    * `--initdb` - This tells `apk` to set-up the database otherwise the following
    errors occur:
        ```
        ERROR: Unable to lock database: No such file or directory
        ERROR: Failed to open apk database: No such file or directory
        ```
    * `--allow-untrusted` - This is a quick and dirty hack to avoid the issues that
        the packages won't be trusted due to the lack of signatures. Ideally, they
        would be pulled out from the `ostree` too.
        ```
        `UNTRUSTED signature` happens
        ```
    * `--logfile=no` - prevents the creation of `/var/log/apk.log`
3. Commit the package as its own branch.
   ```sh
   ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64_curl --tree=dir=pkg.curl --subject "Adding curl."
   ```
   It would be nice to include the version number in the subject, but that is
   more of a nice to have.
4. Lets create new branch for teh base system, ideally when populating it from the rootfs this would have been the
   branch created.
   ```sh
   ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64_base --tree=ref=$(ostree --repo=alpine_repo rev-parse alpine-v3.22.2-x86_64)
   ```
5. Generate the union of it.
    ```sh
    rm -rf alpine-build
    ostree --repo=alpine_repo checkout -U --union alpine-v3.22.2-x86_64_base alpine-build
    ostree --repo=alpine_repo checkout -U --union alpine-v3.22.2-x86_64_curl alpine-build
    ostree --repo=alpine_repo commit --branch=alpine-v3.22.2-x86_64 --link-checkout-speedup alpine-build
    ```
6. Confirm contents
    ```sh
    ostree --repo=alpine_repo ls alpine-v3.22.2-x86_64 /usr/bin
    ```
    This should list all the busybox tools plus `curl`.

The original idea for the first step was convert the output from `ostree cat`
into `--repository <path>` but that has the problem of needing to deal with
commented out lines, not that there should be any in the minirootfs but if
later steps modify it then best to make sure it still works.

My concern with this approach is the `pkg.curl/etc/apk/world` file will simply
contain `curl` and that will conflict with the root filesystem.

Sure enough, that is a problem with the above steps, the resulting
`/etc/apk/world` file only contains `curl` and lacks `alpine-baselayout` and
the other packages from the minimal root file system.

This is as documented from `--union` which is to:
> Keep existing directories, overwrite existing files

## Questions

How to go about creating branches where they start to diverge?
For example, imagining having a common base then splitting it off into server
vs desktop, where desktop branch which has X11 or Wayland installed. From there
you could then imagine branching off so you had a branch for each desktop
environment.

What packages have side-effects in Alpine?
* Is it easy to tell what packages have scripts that run at installation time?
* Is it possible to delay running them until later? Such as running one step
before a final image is made.

## Next
This can't be turned into a bootable image as there is no kernel.

Nominally would need to install `linux-virt`, similar to how curl was installed.

```sh
apk add --root staging-area  --no-cache linux-virt
```

The other thing to consider is `rpm-ostree` , which it takes as input RPMs, and
commits them. This means there could be a `apk-ostree` created which would
take an input APK (Alpine Package Keeper file) and commit them too.

Revisit the second option for installing packages to address the merging files
such as `/etc/apk/world`.

[1]: https://ostreedev.github.io/ostree/buildsystem-and-repos/
