Brief documentation of an attempt I made of importing latest at the time
AlpineLinux into WSL (Windows Subsystem for Linux)

I started with [alpine-minirootfs-3.18.0-x86_64.tar.gz][1] found at
https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/

Or at one of their mirrors.

## Installing AlpineLinux to WSL

```
curl -O https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.0-x86_64.tar.gz
wsl --import Alpine318 C:\ProgramData\WSLDistroStorage\alpine318 alpine-minirootfs-3.18.0-x86_64.tar.gz
wsl -d Alpine318
```

## Add user account and make default
Create a user account and then set the default user to that account, rather
than root.

See Alpine's documentation for [Setting up a new user][2] for a more detailed
account what this is starting to do.
```
adduser donno
# Follow the onscreen prompts.
```

Next set-up auto-login to that account for WSL:
```
vi /etc/wsl.conf
```
Within the file add the following replacing donno for the username you used
above.
```
[user]
default=donno
```
See this [superuser post][2] for more details and alternative methods.

As a reminder for vi use, press escape if you need to stop the current
command then `:wq!` (for save, quit and no confirmation).

For this to take effect the distro needs to be restarted:
`wsl --terminate Alpine38` then when you next launch it you will be logged
in as the user configured.

Once logged in type `whoami` as AlpineLinux's default prompt doesn't include
the current username.


## Bonus tidbit
The filesystem of the WSL distribution can be accessed through Windows
Explorer via: `\\wsl.localhost\Alpine318`, that is to say via
`\\wsl.localhost\<name>` provided the distribution is running.

The IP address of the host can be queried with:
```sh
ip route show | grep -i default | awk '{ print $3}'
```
This snippet is from the [WSL Networking][4] page.

## ALpine 3.22 update

Differences
* Created it on a different drive
* Created it with a fixed size disk image.

```
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.0-x86_64.tar.gz
wsl --install --fixed-vhd --vhd-size 10G --location D:\vms\wsl\alpine322 --name "Alpine 3.22" --from-file alpine-minirootfs-3.22.0-x86_64.tar.gz
```

```sh
dh -h
/dev/sdf                  9.7G      8.7M      9.2G   0% /

# I installed the following packages:
apk add openrc openrc-user nano
```

[1]: https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.0-x86_64.tar.gz
[2]: https://wiki.alpinelinux.org/wiki/Setting_up_a_new_user
[3]: https://superuser.com/a/1627461
[4]: https://learn.microsoft.com/en-us/windows/wsl/networking
