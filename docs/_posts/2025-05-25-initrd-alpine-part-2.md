---
layout: post
title:  "initrd for Alpine - Part 2"
date:   2025-05-25 23:00:00 +0930
---

This is a follow-up my previous post [initrd for Alpine][0] where the project
is extended to allow the user accounts and packages to be customised.

## User accounts

The two requirements for creating user accounts was that the name of the
account should be provided and the SSH key. The starting point for this
was defining a file that is the source of this information that can be
provided.

### Input file

```
<username-0> <ssh-key-0>
<username-1> <ssh-key-1>
...
<username-n> <ssh-key-n>
````

The benefit of this set-up is due to the SSH keys being public, it
means there is minimal downside of sharing the users file (The
one problem there would be if the private keys were exposed now
someone knows what the corresponding username is).

The downside of this set-up is there no provision for providing a
password, nor is it possible to set the GECOS field.

### Parsing file

The easy solution would be to switch to Python, however in the interest on
trying to keep the tools required to build this low and to learn something
new instead write it in the [AWK][1] programming language. This is because
`busybox` includes `awk` so the project can run on a very minimal Alpine
system.

The AWK program is responsible for parsing the input file and generating
the commands to run to create the user and set-up the `authorized_keys`
file for SSH access, so the result the script needs to be run through
`sh`.

The completed AWK program is as follows:
```awk
#!/usr/bin/awk -f
# AWK Script to process the users file
#
# Test with either:
# $ awk -f process-users.awk users > /rootfs/new-users
# $ ./process-users.awk users
#
# Format:
# [username] [public SSH key]`
{
    st = index($0," ")
    substr($0,st+1)
    sshkey = substr($0,st+1, length($0)-st-1)
    print "adduser -D " $1;
    print "mkdir -p /home/" $1 "/.ssh && echo \"" sshkey "\" >> /home/" $1 "/.ssh/authorized_keys"
    print "chown -R " $1 ":" $1" /home/" $1 "/.ssh"
    print "chmod 700 /home/" $1 "/.ssh"
    print "chmod 600 /home/" $1 "/.ssh/authorized_keys"

    # Generate fake password.
    #
    # The alternative to this may be to set sshd_config to UsePAM yes and
    # install openssh-server-pam, which allows the accounts to be locked due
    # to having no password but allow ssh access with the keys.
    t1 = rand() * systime(); t2 = rand() * systime()
    fakepwd = sprintf("%x%x", t1, t2)
    print "echo -e \"" fakepwd "\\n" fakepwd "\" | passwd " $1
}
```

The steps are:
1. Find the first space that separates the name of the user and the start of
   the SSH key.
2. Extract the username
3. Extract the SSH key from after the space to the end of the line.
4. Output the command to create the user (`adduser -D <username>`)
5. Output the command to create the user's home directory, `.ssh` folder within
   and write the key to the `authorized_keys`.
6. Fix-up permissions of the `.ssh` directory and files.
7. Generate fake password - this is because otherwise the account will be
   locked.
8. Take the resulting script and run it through `sh`.

### Putting it together
Since it requires the SSH keys, the assumption is it only makes sense to enable
the users when networking is enabled.

The AWK program is executed and output the shell script it generates into the
new root file system. From there `chroot` into the root file system and run
the script. Lastly, clean-up the shell script by removing it afterwards.

```sh
if [ -f users ]; then
  if [ "$ENABLE_NETWORKING" -gt 0 ]; then
    echo "Adding users."
    awk -f process-users.awk users > /rootfs/new-users
    chroot /rootfs /bin/sh /new-users
    rm /rootfs/new-users
  else
    echo "Adding users relies on SSH keys, so doesn't allow interactive login."
  fi
fi
```

### Using it

To test it with [QEMU][2].
* To enable networking `-net nic -net user`
* Forward port 22 (SSH) to 2222 on the host: `-net user,hostfwd=tcp::2222-:22`

Things didn't work the first time as without setting a password the user account
is locked which prevents the user from loginning in via SSH. To help diagnose
that the syslog service needed to be started with `rc-service syslog start` and
then the log would be looked it via `tail -f /var/log/messages`.

## Custom packages

In support of this the idea was to introduce the concept of "flavours" to the
script, where flavours have different packages. The flavours it would come with
would be `minimal`, `plain` and `standard`.

* minimal is alpine-base split into its part.
* plain is alpine-base
* standard is plain with util-linux, grep, nano and tmux

Additional flavours could be created by creating a new file with one package per
line called `packages.<flavour-name>`. To test this working teh `standard`
flavour was reworked so instead of the package list being hard coded it would
use this approach.

### Input file
* `packages.standard`
````
util-linux
grep
nano
tmux
```

### Using the file
This simply uses `xargs` to read the file and convert it into a single line
which can then be passed to `apk`.

```sh
if [ -f "packages.$FLAVOUR" ]
then
  xargs -a "packages.$FLAVOUR" apk --arch "$ARCH" -X "$BASE_URI/main/" --root /rootfs --initdb --no-cache --allow-untrusted add $PACKAGES
fi
```

What isn't possible is to set-up additional repositories such as community and
edge, but if you could then the packages file could contian `package-name@edge`
to install packages not in the main package repository. It was only writing this
post where I realised I didn't have a way to handle that.

## Configuration script
After installing the packages the next thing to do was to allow a flavour to
run a script to do extra configuration of things that aren't packages or users.

The showcase for this feature was simply to customise the MOTD.
To make the scripting part necsararcy and for it to be useful for doing other
things it was imporant not to use a pre-made file.

If the file was pre-made then the solution for having a script would have been
the wrong solution and the better solution would have been to set-up a folder
that would simply copy its contents into the the root file system. For example,
you create `overlay.<flavour>` directory with `overlay.<flavour>/etc/motd` and
the initrd script could know to copy that over.

To make the MOTD dynamic the idea is to include the name of the distribution
with the version in it.

### Input file
`configure.outside.motd.sh`
```sh
#!/bin/sh
# This is an example of the configure.outside script for the "motd" flavour
# which simply overwrites the message of the day that is baked into the
# image.
ROOT="$1"

. "$ROOT/etc/os-release"

cat << EOF > "$ROOT/etc/motd"
Welcome to $PRETTY_NAME!

This is a simple demonstration to test the ability to configure the instance
with a script.
EOF
```

### Putting it together
```sh
# Perform configuration, outside the rootfs.
if [ -f "configure.outside.$FLAVOUR.sh" ]
then
  sh "configure.outside.$FLAVOUR.sh" /rootfs
fi
````

By performing the configuration outside the rootfs this means the script will
be run by the host system not by `/bin/sh` in the rootfs. This in turn means the
script is going to work if the host is `x86_64` but the rootfs is `aarch64`.

To set-up additional services in the outside script, you need to use the
`ln -sf` approach rather than running `rc-update add`.

However for completeness the ability to provide a configure script that runs
inside the rootfs was supported as well by doing the following:
```
if [ -f "configure.inside.$FLAVOUR.sh" ]
then
  cp "configure.inside.$FLAVOUR.sh" /rootfs
  chroot /rootfs /bin/sh "/configure.inside.$FLAVOUR.sh"
  rm "/rootfs/configure.inside.$FLAVOUR.sh"
fi
```

## Conclusion
This weekend the following features were added:
* Setting-up users that can SSH in
* Defining flavours which can:
  * Provide a list of packages to install.
  * Provide a script to run outside the roofs to perform flavour specific
    configuration.
  * Provide a script to run inside the roofs to perform flavour specific
    configuration.

This took a lot longer to achieve simply because of choosing to write the
scripts as a shell with awk for running with Busybox. The entire system
would have been quicker to write and easier to follow if it was Python
with [TOML][3] (Tom's Obvious Minimal Language) for the configuration aspect.

Update: Alpine 3.22 was released a week later and I didn't end up using this
project to replace my old 3.19 VM with a new one. The approach taken instead
was to simply reinstall it from the ISO. Maybe next time.

[0]: {% post_url 2025-05-22-initrd-alpine %}
[1]: https://en.wikipedia.org/wiki/AWK
[2]: https://qemu.org
[3]: https://toml.io/en/
