---
layout: post
title:  "runc idea - part 2"
date:   2025-11-30 23:10:00 +0930
---

This continues the container idea using runc started in [part 1](0) where the
idea was to turn container images into applications or more likely services.
Last time, the OCI image was turned into an OCI bundle that was runnable with
runc. The problem was the networking wasn't configured to allow the host to
talk to the service running in the container.

The starting point is to consider the Container Network Interface (CNI), which
has the tag line of "networking for Linux containers". Other ideas is to check
to see if [iptables](2) or [nftables](3) (which is the intended as the
replacements for the former tools). The question I would like answered is does
`iptables` or `nftables` enable us to forward ports from the localhost interface
within the container to the host.

## Re-starting

When starting this one, I reran the commands from part 1 on a new fresh install
of Alpine Linux except I was a regularly user this time instead of root.
This proved to be problematic as while /opt/containers was opened by the
regular user, `umoci` would fail to unpack there with the following error:
```
   • umoci encountered a permission error: maybe --rootless will help?
   x create runtime bundle: unpack rootfs: chown rootfs: lchown /opt/containers/valkey-9.0.0/rootfs: operation not permitted`
```

By adding the `--rootless` argument to `umoci` what it does is it doesn't try
to change the owner to `root` as it is in the layer tarballs and therefore
allows it to be created.

The [documentation](4) of rootless option of `umoci` mentions
> umoci also supports the user.rootlesscontainers specification, which allows for further emulation of things like chown(2) inside rootless containers using tools like PRoot.

That sounds as if it may be a solution to the problem where the valkey
container wants to change the owner (run `chown`) of everything to valkey, so
that needs to be looked into later.

When running the container, since it tried to do the chown stuff that still
failed but using an existing user ID also failed such as 5 for `sync`.

```
ERRO[0000] runc run failed: unable to start container process: error during container init: unable to setup user: cannot setgid to unmapped gid 5 in user namespace
```

Possibly need to run `newgidmap` or `newuidmap`, but unsure, in the end I set
the uid and gid back to 0 and simply modified
`rootfs/usr/local/bin/docker-entrypoint.sh` to skip the `chown` and it ran.

This time the networking worked, however before testing connecting to the
server, a problem was discovered. Starting a second container failed because
it said the port was already in use, which suggests the networking isn't in its
own namespace. Running the `valkey-cli` worked, at first I thought it was 
because it was installed on the host rather than using the `chroot` approach
done last time but both that and the `chroot` worked.

I then when back to the original environment and sure enough under that I could
run two `valkey-server` on same port via `runc run vk-3` and `runc run vk-1`.
This indeed worked and the second run didn't conflict with the port in the
first.

The difference was the original system was Wolfi where the second system was
Alpine 3.22. Comparing the OCI runtime specification for the two systems,
in the Wolfi one it has `network` namespace listed but the Alpine one didn't,
instead it has the type `user` likely because of the `--rootless`.

## Back to Root
Re-starting but as `root` this time, which brought it back to having a network
namespace and thus was able to start three containers of Valkey.

Testing using `nsenter`, to run `valkey-cli` within the container. The first
step is to find the PID of the `valkey-server` via `ps`, then enter the
namespace of that process with `nsenter -t <pid> valkey-cli`. This raised an
error `Could not connect to Valkey at 127.0.0.1:6379: Connection refused`. Try
again this time add `-n` argument to `nsenter` which causes it to enter the
network namespace and that works. This experiment provides a new idea, use
`socat` to essentially folder data to the namespace.

Now this has brought us back to base where networking is namespaced, can look
at the networking options.

## Networking

### socat

For this, we need the port that the service is running on as it only forwards
the port. In the example with Valkey 9.0.0 this is port 6379.

In this case, nsenter is used to run the command in the same network namespace 
as the Valkey process.
```sh
nsenter -t $(runc state vk-2 | jq ".pid") -n socat UNIX-LISTEN:/tmp/socket TCP:127.0.0.1:6379
```
The reason `runc exec` can't be used here is it would require `socat` exist
within the root file system of the container. This command therefore won't work
unless `socat` was added to the `rootfs` directory in a directory included in
`$PATH`.
```sh
runc exec vk-3 socat UNIX-LISTEN:/tmp/socket TCP:127.0.0.1:6379
```

Then run the following command which listens on the same port on the host and
connects to the socket which is connected ot the port in the network namespace.
```sh
socat TCP-LISTEN:6379 UNIX:/tmp/socket
```

Now `valkey-cli` on the host will connect to that service, as it the default
host and port.

* Pros
    * Specific single port - only get the individual thing
    * The port could come from the `org.opencontainers.image.exposedPorts`
      annotation.
* Cons
    * Do need to know the port.

This approach does seem easy to automate, especially with the hooks.

### iproute2
First things to account for is the `ip` command from BusyBox does not have the
`netns` subcommand, so `iproute2` is needed to peruse that.

Special thanks to Murat Kilic [article][6] about network setup with runc and
a scripted approach by svv on [Serverfault][7], which I found shortly after
following Murat's post. This creates virtual ethernet device with one end in the
network namespace for the container and the other outside (i.e. on the host).
The downside to this approach is creating this device requires root privileges.

```sh
ip netns add ctr_valkey_9.0.0
ip link add name veth-host type veth peer name veth-ctr-valkey
ip link set veth-ctr-valkey netns ctr_valkey_9.0.0
ip netns exec ctr_valkey_9.0.0 ip addr add 192.168.10.1/24 dev veth-ctr-valkey
ip netns exec ctr_valkey_9.0.0 ip link set veth-ctr-valkey up
ip netns exec ctr_valkey_9.0.0 ip link set lo up
ip link set veth-host up
ip route add 192.168.10.1/32 dev veth-host
ip netns exec ctr_valkey_9.0.0 ip route add default via 192.168.10.1 dev veth-ctr-valkey
```

1. Create a new named network namespace - this is naming it after the container.
2. Create a virtual ethernet adapter - one end represents the host the other
   the container.
3. Move the container end to the network namespace.
4. Set-up IP address for the container side of the ethernet adapeter.
5. Bring up the network interface for the container (network namespace.)
6. Bring up the network interface for the host side.
7. Route traffic.

The `ip netns exec` are running the command that follows in a network namespace,
it could probably be done with `nsenter` instead, but it takes the network
namespace name rather than PID. However, I suspect with a container hook this
set-up can become part of that.

Checking the working, by trying to ping the IP of the container from the host
so in this case that is `192.168.10.1`.

You can also `ip -netns ctr_valkey_9.0.0 link ls` and
`ip -netns ctr_valkey_9.0.0 addr` to help troubleshoot.

The link appears as;
```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth-ctr-valkey@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether de:b0:21:43:e3:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

The address is:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host proto kernel_lo
       valid_lft forever preferred_lft forever
3: veth-ctr-valkey@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether de:b0:21:43:e3:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.1/24 scope global veth-ctr-valkey
       valid_lft forever preferred_lft forever
    inet6 fe80::dcb0:21ff:fe43:e380/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```

**Using it**

In the `config.json`, go to  `linux.namespaces` and find the namespace with
type of `network` and add the path property to it i.e.
```json
    { 
        "type": "network",
        "path": "/var/run/netns/ctr_valkey_9.0.0"
    }
```

Re-run the container and now `valkey-cli -h 192.168.10.1` from the host works..

Of course, now since the namespace has a particular path it means you can't run
the container more than once.

#### Automating

I didn't get time for this avenues.

## Unexplored avenues

I'm not sure if `iptables` or `nftables` could be used to set-up the port
forwarding between namespaces.

For non-network avenues, explore the `user.rootlesscontainers` and PRoot
mentioned by [umoci's](4) documentation about rootless.

The other thing that may have been useful but wasn't needed in today's session
was:
```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
````

[0]: {% post_url 2025-11-28-runc-idea-part-1 %}
[1]: https://www.cni.dev/
[2]: https://www.iptables.org/
[3]: https://wiki.nftables.org/
[4]: https://umo.ci/quick-start/rootless/
[5]: https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking
[6]: https://medium.com/@Mark.io/https-medium-com-mark-io-network-setup-with-runc-containers-46b5a9cc4c5b
[7]: https://serverfault.com/a/1178962
