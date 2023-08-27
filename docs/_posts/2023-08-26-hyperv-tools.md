---
layout: post
title:  "Hyper-V Tools"
date:   2023-08-26 13:00:00 +0930
---

Today, I came across some tools for Hyper-V when looking into
[Host Compute System](0) APIs for Microsoft Windows for controlling
virtual machines. This covers more system administration tools rather than
the programming interface.

## hcsdiag

hcsdiag.exe is the Hyper-V Host Compute Service Diagnostics tool and the first
of the two tools I looked at.

The main command it offers is **list** which lists running containers and
virtual machines.

On my system it shows a VM for WSL and VMs from Hyper-V labelled as VMMS for
**Virtual Machine Management Service**. It can list the VM used for the Windows
Sandbox feature with the name **Madrid**.

Features
* Run a process inside the container.
* Launch an interactive console inside the container.
  This doesn't work for VMMS or WSL.
* Sharing a host folder with the container (`hcsdiag share`)
* Sharing a virtual hard disk file into the container `vhd share`)
* Terminating a running container (`hcsdiag kill`)
* Reading/writing to and from the container. This is sounds interesting and
  something I would want to try out to see how useful it is.

## HVC

hvc.exe is the Hyper-V Command Line Tool and is included with Windows for
managing virtual machines from the command line. At the time of writing
the Windows 10 version that I had didn't include the ability to create a VM.

For creating a virtual Machine in Hyper-V, the best way to do that from the
command line is with the [New-VM](1) command-let in PowerShell. However, you
will likely want to use the [New-VHD](2) command-let to create a virtual
hard-drive image for the new VM, however the former command has two parameters
for creating a new image (`-NewVHDPath` and `-NewVHDSizeBytes`)

### Listing Virtual Machines
`hvc list`

This lists the status and names of the virtual machines from Hyper-V.

The common status are off, running, saved and paused as the others are
transitions.

Possible states (its status)
    * off - The VM is powered off.
    * on - The VM is powered on.
    * starting - The VM is powering on.
    * stopping - The VM is powering off.
    * saved - The VM has been saved.
    * paused - The VM has been paused.
    * resetting - The VM is being reset.
    * saving - The VM is saving.
    * pausing - The VM is pausing.
    * resuming - The VM is resuming.

There are flags for different style options:
* `-i` will include the virtual machine ID (UUID)
* `-r` will filter it to just the running VMS
* `-q` will only print the the name (or if -i used the IDs) with no headings.

## Starting VM

Starting a virtual machine is simple once you know the name.

`hvc start <VM-Name>`

The PowerShell equivalent would be:
`Start-VM -Name <VM-Name>`
With the advantage of PowerShell if you give it -PassThru it will return back
the VirtualMachine object ready for-use by another command.

## SSH
`hvc ssh [options] [user@]<VM>`

This interested me a lot as I previously set-up a PowerShell command-let to
start a VM ([Start-VM](3)) then query the IP address of that VM before
issuing a ssh to the VM.

This was in part because Hyper-V uses DHCP and doesn't seem to have a nice way
of setting a dedicated IP address to a given VM say by the MAC address.

I booted up a virtual machine then tried the command it it kept failing.

```
The ssh connection to the VM failed. Is the ssh service configured in the
virtual machine?
kex_exchange_identification: Connection closed by remote host
```

I added -v and it passed the command on to OpenSSH client whch printed where it
was looking for the  identity file but more importantly the proxy command it
was executing.

```
C:/WINDOWS/system32/hvc.exe nc -t vsock,ip --ssh --host-prefix hyper-v/ "hyper-v/alpine311" 22
```

My first thought was it was strange it wasn't working as it seemed like it was
able to map directly from port 22 inside the VM to the host. After a couple
of web searches the issue was it needed the Hyper-V tools installed in the VM
to function. Given I was using Alpine, meant installing `hvtools` and starting
the daemons, which I cover later.

With the tools running, the command worked from the host.
From there I did a quick benchmark and observed that the special Hyper-V method
was about 20MB/s faster.

For my most common virtual machine, I tweaked my SSH configuration to have a
entry in which had the ProxyCommand for the VM and defaulted to the user, so
can go straight through ssh.exe rather tha hvc.exe first.

## SCP
`hvc scp [options] [[user@]<VM>:]file1 [[user@]<VM>:]file2`

Handy if you want to copy a file across and don't want to look-up the IP
address.

It is a shame that Microsoft didn't have an option to set-up VM in DNS as say
VM-Name.vm.local, such that when they the virtual network adapter allocates the
IP that it adds the corresponding entry for the host to use.

## Other tricks

* Start the Virtual Machine connection to the VM.
    ```
    hvc console <VM-Name>
    ```
    This saves the hassle of opening Hyper-V Machine and double clicking the
    virtual machine to connect to it.

    The alternative is `vmconnect localhost <VM-Name>` which if you only type
    `vmconnect` will open a window that looks similar to the Remote Desktop
    Connection (`mstsc.exe`) but with option to select or type the name of
    a Virtual Machine.

* For a list of all the Hyper-V commands in Powershell
  ```
  Get-Command -Module hyper-v | Out-GridView
  ```

## Tool Installation

The tools required to provide this interaction are from source repository for
the Linux kernel and are in the [tools/hv](4) directory.


* kvp daemon is responsible for what they call the key value pair
  functionality. In practice what this means is communicating configuration
  settings between guest and host such as its IP address and the host's
  machines name hosts versions.

  For example, there is a key HostingSystemEditionIds which for my machine
  had the value of 48 which corresponds with  Windows 10 Pro  from
  the documentation for [GetProductInfo](5).

  A browse of it source suggests it enables the host to query the contents of
  /etc/os-release, however I was unable to figure out how to query it.
  Some documentation suggested it was accessible via Windows Management Instrumentation (WMI).
* fcopy daemon is responsible for host to guest copy functionality.
* vss demon is responsible for host initiated guest snapshot for Hyper-V.

I later found the [Hyper-V Integration Services](6) documentation which
explains the daemons as well.

### Alpine

Install the package then start the services.
```sh
apk add hvtools
rc-service hv_fcopy_daemon start
rc-service hv_kvp_daemon start
rc-service hv_vss_daemon start
```

To ensure the services are started the next time they boot run these commands:
```sh
rc-update add hv_fcopy_daemon
rc-update add hv_kvp_daemon
rc-update add hv_vss_daemon
```

## Debian / Ubuntu

```sh
apt install hyperv-daemons
```

## Red-hat / Fedora

```sh
dnf install hyperv-daemons
# Or if older:
yum  install hyperv-daemons
```

[0]: https://learn.microsoft.com/en-us/virtualization/api/hcs/overview
[1]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/new-vm
[2]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/new-vhd
[3]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/start-vm
[4]: https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/tools/hv?h=v6.4.12
[5]: https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getproductinfo
[6]: https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/reference/integration-services