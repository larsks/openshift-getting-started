# Project 1: Creating virtual machines

Ultimately, we're going to be installing OpenShift onto some virtual
machines. Before we can do that, we need to understand how to create
and manage virtual machines under Linux.

## The tools

You are going to be interacting primarily with two command line tools:

- [`virt-install`][virt-install] helps create virtual machines
- [`virsh`][virsh] provides a set of commands for managing virtual machines

[virsh]: https://libvirt.org/manpages/virsh.html
[virt-install]: https://www.mankier.com/1/virt-install

## Tasks

### Download virtual machine images

We'll need to get an operating system onto our virtual machine. There
are a few ways to do that (for example, one can just run a regular OS
installer), but for our purposes it's much more convenient to start
with a pre-built image.

Most distributions provide "cloud images", so called because they're
designed for use in a cloud environment like AWS, OpenStack, etc. It
turns out they also work great for local use. For example, you can
look at the cloud images provided by [CentOS][centos-images] and
[Fedora][fedora-images].

[centos-images]: https://cloud.centos.org/centos/
[fedora-images]: https://alt.fedoraproject.org/cloud/

While `libvirt` can use images anywhere on your filesystem, it helps
if we put them in a directory that libvirt is configured to use as a
"[storage pool][]". The default location for images (when interacting with
`libvirt` running  as `root`) is `/var/lib/libvirt/images`.

[storage pool]: https://libvirt.org/storage.html

Download the [CentOS 8 Stream cloud image][centos-8-stream]
to `/var/lib/libvirt/images`. You should end up with a file named
something like `CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2`.

[centos-8-stream]: https://cloud.centos.org/centos/8-stream/x86_64/images/CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2

Make sure `libvirt` is aware of the new image by running:

```
virsh pool-refresh default
```

You should now see the image in the output of `virsh vol-list
default`:

```
$ virsh vol-list default
 Name                                                   Path
--------------------------------------------------------------------------------------------------------------------------------------
 CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2   /var/lib/libvirt/images/CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2
```

### Create a virtual machine

We'll use the `virt-install` command to create a new virtual machine.
Run the following command:

```
virt-install \
  -n centos-example \
  --memory 8192 \
  --os-variant centos8 \
  --network network=default \
  --disk pool=default,size=20,backing_store=CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2,backing_format=qcow2 \
  --import \
  --cpu host-passthrough \
  --noautoconsole
```

Let's walk through those command line options:

- `-n centos-example` sets the name of the virtual machine
- `--memory 8192` gives the vm 8GB memory
- `--os-variant centos8` provides helps `virt-install` choose some
  default devices (such as the type of disk controller, network
  interface, etc) which otherwise we would need to specify explicitly
- `--network network=default` tells `virt-install` to connect the
  virtual machine to the "default" libvirt network (read more about
  that below in "[A note about libvirt
  networking](#a-note-about-libvirt-networking)")
- `--disk ...` configures our boot disk:
  - `pool=default,size=20` tells `virt-install` to create a 20GB disk
    in the `default` pool (i.e., in `/var/lib/libvirt/images`).
  - `backing_store=...,backing_format=qcow2` tells `virt-install` to
    create a [copy-on-write][cow] image. That means it will start out
    sharing the same content as the backing store, but any writes will
    go to a new file.
  - `--import` means "we're not running an installer, just boot from
    this disk".
  - `--cpu host-passthrough` makes the processor in the virtual
    machine match the processor on the host (this has performance
    benefits, but can limit your ability to live migrate a virtual
    machine between hypervisors if they don't have identical
    processors).
  - `--noautoconsole` means don't try to automatically open a GUI
    console for this machine when it starts (because we're logged in
    remotely so this probably wouldn't work).

[cow]: https://en.wikipedia.org/wiki/Copy-on-write

In a few seconds, running `virsh list` should show the new virtual
machine:

```
$ virsh list
 Id   Name             State
--------------------------------
 61   centos-example   running
```

### A note about libvirt networking

We've attached a virtual machine to the libvirt "default" network.
This is a private network internal the host on which your virtual
machine is running. When your machine starts up, libvirt creates a
virtual network interface and attaches it to the `virbr0` bridge
device. You can inspect the configuration of that device like any
other network interface:

```
$ ip addr show virbr0
10: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:87:f6:3c brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
```

From this you can see that all virtual machines on this network will
have addresses on the `192.168.122.0/24` network.

The virtual machine we've created in this step isn't connected
directly to the public internet, but it does have *outbound* access
thanks to a firewall configuration implemented by `libvirt`.

The libvirt wiki has more information about [libvirt networking][].

[libvirt networking]: https://wiki.libvirt.org/page/Networking

### Remove a virtual machine

Yay, you have a virtual machine running! So, what can you do with it?
At this point, not much, because we've skipped an important step: you
won't be able to log into this virtual machine because cloud images are
configured without any sort of default password (this is a security
measure that prevents problems when someone boots one of
these images when connected to the public internet). Let's destroy
this virtual machine and configure things properly.

There are two steps to removing a virtual machine:

1. Stop the virtual machine from running.
2. Remove the virtual machine definition from `libvirt`

To stop the virtual machine, run:

```
virsh destroy centos-example
```

This is effectively the same as pulling the plug on a real machine:
the virtual machine will stop running immediately, but the definition
still exists so we could boot it up again with the `virsh start`
command.

To remove the definition from libvirt *and* remove the boot disk, run:

```
virsh undefine --remove-all-storage centos-example
```

### Create a virtual machine that we can actually use

First, make sure your ssh public key is available on the hypervisor.
I'm assuming you've saved it in a file in the current directory named
`id_rsa.pub`, but you're free to name it something else.

Now, let's re-create our virtual machine, but this time we're adding an
additional command line option:

```
virt-install \
  -n centos-example \
  --memory 8192 \
  --os-variant centos8 \
  --network network=default \
  --disk pool=default,size=20,backing_store=CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2,backing_format=qcow2 \
  --import \
  --cpu host-passthrough \
  --noautoconsole \
  --cloud-init ssh-key=id_rsa.pub
```

The `--cloud-init ssh-key=id_rsa.pub` causes `libvirt` to attach a
virtual CD-ROM device to your machine that contains some configuration
information. This is read by [cloud-init][] when the system boots.
Cloud-init is a tool installed on most "cloud" images that is designed
to configure a system using metadata available from the cloud
environment (which means this only works if you're booting an image
that is configured to run `cloud-init`).

[cloud-init]: https://cloudinit.readthedocs.io/en/latest/

This will configure your ssh public key in the `authorized_keys` file
for the `root` user.

### Log in to your virtual machine

To log in to your virtual machine, you'll first need to find its
address. We can get that information with the `virsh domifaddr`
command:

```
$ virsh domifaddr centos-example
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet70     52:54:00:b8:05:90    ipv4         192.168.122.68/24
```

With that information, we can log in to the virtual machine as `root`:

```
$ ssh root@192.168.122.68
Warning: Permanently added '192.168.122.68' (ED25519) to the list of known hosts.
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Thu Dec  2 00:42:22 2021 from 192.168.122.1
[root@localhost ~]#
```

Once you've logged into your virtual machine, you can verify that you
have outbound connectivity by trying to connect to an outside
resource. For example:

```
# curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

If you're curious about how `cloud-init` works, you can see the
configuration data provided by libvirt by mounting the virtual CD-ROM device:

```
# mount -o ro /dev/sr0 /mnt
# ls /mnt
meta-data user-data
```


## Exercises

This section is your chance to take what you've learned in the
previous section and expand upon it a bit. The following two tasks are
meant to be largely self-directed; see how far you can get.

### Scripting things

Write a script `create-machines.sh` that will let you create *N*
virtual machines with a single command. Running something like...

```
sh create-machines.sh 3
```

...should create three virtual machines named `node0`, `node1`, and
`node2`.

Write a second script `destroy-machines.sh` that will destroy the
virtual machines created by the earlier script.

If you want to get extra fancy, modify the `create-machines.sh` script
so that you can specify the amount of memory and number of cpus to
assign to virtual machines when running the script.

### GUI access

There is a GUI available for managing virtual machines called
[`virt-manager`](https://virt-manager.org). Virt-manager is able to
manage virtual machines on a remote host. You can install this on
RHEL, Fedora, and CentOS by running `yum install virt-manager`.

As an exercise, figure out how to get `virt-manager` connected to
`dev.massopen.cloud` so that you can view the system console for your
virtual machines.
