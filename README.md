# QEMU setup w/ network

A simple guide on how to set up a vm w/ QEMU for kernel development.

## Configuring/Building the kernel

First download the kernel. You can either get the latest through:
```bash
$ git clone https://github.com/torvalds/linux
```
or just grab one from [here](https://www.kernel.org/).

Navigate to the kernel directory and type:
```bash
$ make defconfig
```
This command will create a ***.config*** file with default options.
Now its time to configure our kernel. Type
```bash
$ make menuconfig
```
This command will prompt you to a console gui where you can select what options
you want for your kernel. In our case we would like to select all **VIRTIO** options. You can find all the corresponding values by using the search command on the menuconfig window:
```bash
/VIRTIO
```
You will propably want to enable all the shown options. If you are not sure whether
you should enable an option or not, just enable it, the chances of you breaking 
something are (hopefully) low. If you still need some guidance you can take a look
[here](https://www.linux-kvm.org/page/Tuning_Kernel).

Now its time to compile the kernel
```bash
$ make
```
This may take some time so be patient.

In case something goes wrong, I have included two **.config** files for two different kernel versions respectively.

Finally our kernel is ready to use.

## Creating a rootfs

Create a .img file with the dd utility
```bash
$ dd if=/dev/zero of=rootfs.img count=0 bs=1 seek=1G
```
This command will create a 1 gigabyte file named rootfs.img
```bash
$ mkfs.ext4 roots.img
```
and this one will convert rootfs.img to an ext4 file system.
Now its time to bootsrap a basic debian system to rootfs.img. We are going to use
the bootrstrap utility
```bash
$ mount rootfs.img /mnt
$ debootstrap stable /mnt
```
Now we need to setup a root password for this system.
```bash
$ chroot /mnt
$ passwd root
```
and set the password you wish.

Now evertyhing is ready to "boot" for the first time.

## Booting up for the first time

```bash
$ qemu-system-x86_64 	\
  -kernel /path/to/linux/arch/x86/bzImage   \
  -drive file=/path/to/rootfs.img,format=raw \
  -append "net.ifnames=0 root=/dev/sda rw console=ttys0" \
  -enable-kvm
  -cpu host
  -nographic
```

# Network

## Creating the tap interface

We need to create a tap interface to the host machine
```bash
$ ip tuntap add mode tap tap0
```
We need to assign an ip to the tap interface
```bash
$ ip addr add <insert an ip>/24 dev tap0
$ ip addr add 192.168.1.10/24 dev tap0 #example
```
Now, we want the tap interface to be rerouted to our main network interface
```bash
$ iptables -t nat -A POSTROUTING -o <your host net interface> -j MASQUERADE
```
You will also want to enable ip forwarding
```bash
$ echo 1 > /proc/sys/net/ipv4/ip_forward
```
You can make the last change permanent by editing the above file with superuser privileges.

The above setting may change/delete the default dns servers. You can resolve this
issue by editing the **/etc/resolv.conf** file by adding the following lines at the end of the file
```bash
nameserver 8.8.8.8
nameserver 8.8.4.4
```
Your tap interface is now ready to be used.

## Guest Network

You need to assign an ip to the network interface of your guest in qemu.
You can inspect the network interfaces with the command:
```bash 
$ ip a
```
Most of the times the name of the network interface is **eth0**.
So we type on the qemu/guest:
```bash
$ ip addr add <insert an ip>/24 dev eth0
$ ip addr add 192.168.1.11/24 dev eth0 #example
```
If you wish to create a static ip you will have to edit the **/etc/network/interfaces/** file and add the following at the end of the file
```bash
auto eth0
iface eth0 inet static
	address <ip of eth0-guest network interface>
	netmask 255.255.255.0
	gateway <ip of host's tap interface-host tap network interace>
	dns-nameservers 8.8.8.8 8.8.4.4
```

## Launching qemu w/ network

The only thing that must be done, is to let qemu know that there is a network interface available and a virtual nic for it.
```bash
$ qemu-system-x86_64 	\
  -kernel /path/to/linux/arch/x86/bzImage   \
  -drive file=/path/to/rootfs.img,format=raw \
  -append "net.ifnames=0 root=/dev/sda rw console=ttys0" \
  -enable-kvm
  -cpu host
  -nographic
  -net tap,script=no,ifname=tap0,vhost=on
  -net nic,model=virtio,macaddr=00:d8:61:36:40:b9 #example mac addr
```

And that's it! If everything is done correctly you should have a fully functional vm with access to the internet(install packages, ping servers etc).
