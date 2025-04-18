---
layout: post
title:  "Barebone Alpine Containers"
date:   2025-04-18 22:00:00 +0930
---

Today I can across [Amazon's documentation][0] on their barebones containers,
which uses `dnf` (Red Hat's successor to `yum`) for their Amazon Linux
distribution and so I was inspired  to try it with Alpine Linux and its package
manager `apk`.

For a several months now I have had the idea of exploring the idea of creating
a container image from layers where each layer was created from a distributions
package. The idea there would be each package being its own layer so you can
share layers for the same package between images for different programs for
example a layer with libzstd. That isn't however what is being looked at today.

Alpine Base
-----------
The starting point was reading [Bootstrapping Alpine Linux][1], which shows
a statically linked version of `apk` being used to install the base system.
Typically, the base images for Alpine Linux containers are built from a tarball
containing the root filesystem. That is to say, they don't use this approach.

```Dockerfile
FROM alpine:3.21 AS base
ARG ARCH="x86_64"
ARG PACKAGE="alpine-base"
ADD --chmod=0744 https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.6/x86_64/apk.static /apk.static
# The --allow-untrusted accounts for UNTRUSTED signature.
RUN /apk.static --arch ${ARCH} -X https://dl-cdn.alpinelinux.org/alpine/latest-stable/main/ --root /rootfs --initdb --no-cache --allow-untrusted add ${PACKAGE}

FROM scratch
COPY --from=base /rootfs /
ENTRYPOINT ["/bin/sh"] 
```

There doesn't seem to be an easy way to avoid the `--allow-untrusted` as you
can't provide an explicit signature on the command line. As of this post, the
package depends on 10 other packages so potential of them would be required.

For containers, the `alpine-base` is perhaps a little more than you would need
as it brings in `openrc`. So if you were interested in creating a base container
you may be better off with this packages or at least checking what `alpine-base`
depends on and filtering out what isn't needed yourself.

`alpine-baselayout alpine-conf alpine-release apk-tools busbox busybox-suid musl-utils`

The `alpine-conf` is responsible for proving the setup scripts for installing
and configuring Alpine and extra packages such as sshd so is a candidate for
ignoring.

Single Program
--------------
The program chosen to try out this experiment to see if the `alpine-base` package
can be avoided was `curl`.

The Containerfile for this is essentially same above but PACKAGE would be curl
and the ENTRYPOINT would be `/usr/bin/curl`. It is not easy to configure that at
build time as you can't provide build arguments there as they aren't substituted
at build time.

### Building
```sh
podman build --build-arg PACKAGE=curl -t myalpine:curl
```

### Running
```sh
podman run --rm -it localhost/myalpine:curl https://google.com
```

The output as the non-www version was given such that it is smaller is:
```
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="https://www.google.com/">here</A>.
</BODY></HTML>
```

### Checking
```sh
podman save --format oci-archive --output curl_container.tar localhost/myalpine:curl
```

The complete file list for layer with curl was as follows:
```
dev/
etc/
etc/apk/
etc/apk/arch
etc/apk/world
etc/ssl/
etc/ssl/cert.pem
etc/ssl/certs/
etc/ssl/certs/ca-certificates.crt
etc/ssl/ct_log_list.cnf
etc/ssl/ct_log_list.cnf.dist
etc/ssl/openssl.cnf
etc/ssl/openssl.cnf.dist
etc/ssl/private/
etc/ssl1.1/
etc/ssl1.1/cert.pem
etc/ssl1.1/certs
lib/
lib/apk/
lib/apk/db/
lib/apk/db/installed
lib/apk/db/lock
lib/apk/db/scripts.tar
lib/apk/db/triggers
lib/ld-musl-x86_64.so.1
lib/libc.musl-x86_64.so.1
proc/
tmp/
usr/
usr/bin/
usr/bin/curl
usr/lib/
usr/lib/engines-3/
usr/lib/engines-3/afalg.so
usr/lib/engines-3/capi.so
usr/lib/engines-3/loader_attic.so
usr/lib/engines-3/padlock.so
usr/lib/libbrotlicommon.so.1
usr/lib/libbrotlicommon.so.1.1.0
usr/lib/libbrotlidec.so.1
usr/lib/libbrotlidec.so.1.1.0
usr/lib/libbrotlienc.so.1
usr/lib/libbrotlienc.so.1.1.0
usr/lib/libcares.so.2
usr/lib/libcares.so.2.19.4
usr/lib/libcrypto.so.3
usr/lib/libcurl.so.4
usr/lib/libcurl.so.4.8.0
usr/lib/libidn2.so.0
usr/lib/libidn2.so.0.4.0
usr/lib/libnghttp2.so.14
usr/lib/libnghttp2.so.14.28.3
usr/lib/libpsl.so.5
usr/lib/libpsl.so.5.3.5
usr/lib/libssl.so.3
usr/lib/libunistring.so.5
usr/lib/libunistring.so.5.1.0
usr/lib/libz.so.1
usr/lib/libz.so.1.3.1
usr/lib/libzstd.so.1
usr/lib/libzstd.so.1.5.6
usr/lib/ossl-modules/
usr/lib/ossl-modules/legacy.so
var/
var/cache/
var/cache/apk/
var/cache/misc/
```

Bigger System - gdal
--------------------
Here is an example with the GDAL tools, granted this one may be better if
it didn't set an entry-point such that when you run it you had to provide the
name of the tool as otherwise you have to overwrite it at runtime if you want
to use something other than `gdal_translate`.

```Dockerfile
FROM alpine:3.21 AS base
ARG ARCH="x86_64"
ADD --chmod=0744 https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.14.6/x86_64/apk.static /apk.static
# The --allow-untrusted accounts for UNTRUSTED signature,
RUN /apk.static --arch ${ARCH} -X https://dl-cdn.alpinelinux.org/alpine/edge/main/ -X https://dl-cdn.alpinelinux.org/alpine/edge/community/ --root /rootfs --initdb --no-cache --allow-untrusted add gdal-tools

FROM scratch
COPY --from=base /rootfs /
ENTRYPOINT ["/usr/bin/gdal_translate"]
```

[0]: https://docs.aws.amazon.com/linux/al2023/ug/barebones-containers.html
[1]: https://wiki.alpinelinux.org/wiki/Bootstrapping_Alpine_Linux
