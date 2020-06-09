# Ubuntu netboot

**Work in progress**

## Introduction
This manual is about setting up the server for installing Ubuntu Server 18.04.1 over the network through PXE. 

Some other versions of Ubuntu can be installed using this method too.

## System configuration:
1) Server with two network interfaces - ens0 - for Internet, ens1 - for the local network. The server is used as a gateway of the local network.
2) Machines without OS in local network with PXE netboot OS installation feature. 


## Preparation

In this paper, a `nano` editor will be used, but you can use your favourite editor.

Packages in this paper are installed using `aptitude`.

You will need to configure the installation server, install and configure the following services on it:
- TFTP
- HTTP
- DHCP

All next commands - for the server machine.

### Installing packages and create a folder for installation files

First - get root rights
```
sudo su
```
Next - install services and create a directory.

In this tutorial as DHCP server will be used `isc-dhcp-server` and as HTTP server will be used `apache2`.

Install services:
```
apt-get install aptitude 
aptitude -R install atftpd -y
aptitude install apache2 tftpd-hpa isc-dhcp-server -y
```

The directory `/srv/tftp/ubuntu` will be used to store and distribute files, but another one can be used to.

Create directory for installation files:
```
mkdir /srv/tftp/ubuntu/
```

### Add netboot files

You should mount the image and copy the contents of `./install/netboot/` from the image of the version of your Ubuntu to the distribution directory `/srv/tftp/ubuntu`.

Loading image:
```
wget -O /home/{USER_NAME}/Downloads/ubuntu-18.04.1-server-amd64.iso http://old-releases.ubuntu.com/releases/18.04.0/ubuntu-18.04.1-server-amd64.iso
```

Mounting an image:
```
mount -o loop /home/{USER_NAME}/Downloads/ubuntu-18.04.1-server-amd64.iso /mnt
```
where {USER_NAME} - your username. 

Copy:
```
cp -fr /mnt/install/netboot/* /srv/tftp/ubuntu
```

Unmount an image:
```
umount /mnt
```

## Server configuration:
In this paper, we use a new server without any previous configuration. 

### Configuring the gateway  

You can skip this step if you have other gateway configuration but in next steps expected that:
* the server has IP 192.168.0.1 in the local network
* local network have access to the internet (server as a gateway) 

#### Configure interfaces
First, you need to set the static IP in the local network (we will use 192.168.0.1). In this example, we get IP for eth0 (Internet) by DHCP. 

Modifying the `/etc/network/interfaces` file
```
nano /etc/network/interfaces
```

*File after changes:*
```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
        address 192.168.0.1
        netmask 255.255.255.0
```

#### Configure DHCP for local network
Next step - set eth1 to be used by isc-dhcp-server. 

Modifying the `/etc/default/isc-dhcp-server` file
```
nano /etc/default/isc-dhcp-server
```

*File changes:*
```
INTERFACESv4="eth1"
INTERFACESv6=""
```

Modifying the `/etc/dhcp/dhcpd.conf` file
```
nano /etc/dhcp/dhcpd.conf
```

*File after changes:*
```
# dhcpd.conf

option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;

#default-lease-time 600;
#max-lease-time 7200;

ddns-update-style none;

#authoritative;

option domain-name-servers 8.8.8.8, 8.8.4.4;

option subnet-mask 255.255.255.0;
option broadcast-address 192.168.0.255;

subnet 192.168.0.0 netmask 255.255.255.0 {
    range 192.168.0.10 192.168.0.254;
    option routers 192.168.0.1;
    default-lease-time 604800;
    max-lease-time 604800;
    #filename = "pxelinux.0";
}

```
#### Restart services
Restart networking and dhcp 
```
/etc/init.d/networking restart
/etc/init.d/isc-dhcp-server restart
```

#### Configure Network Address Translation
In this step, you need to enable IPv4 forwarding, add a NAT forwarding rule using iptables, and save iptables configuration. 


Update `/etc/sysctl.conf` file:
```
nano  /etc/sysctl.conf
```
*Uncomment string*
```
net.ipv4.ip_forward=1
```

Next - add a rule to iptables:
```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Save the iptables configuration:
```
sudo apt install iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4
```
Ensure the rules load at boot:
```
nano /etc/rc.local 
```
*Add to file:*
```
/sbin/iptables-restore < /etc/iptables/rules.v4
```

Gateway is configured - reboot system to apply changes
``` 
reboot 
```

And final - check that the Internet is available from the local network. For example, you can use the other machine, connect it for the local network and check internet availability using browser. 

### TFTP server
In this step, you need to change the USE_INETD flag and set correct TFTP_DIRECTORY in configuration files.  

Modifying the `/etc/default/atftpd` file
```
nano /etc/default/atftpd
```

*File changes:*
```
USE_INETD = false
```

Modifying the `/etc/default/tftpd-hpa` file
```
nano /etc/default/tftpd-hpa
```

*File changes:*
```
TFTP_DIRECTORY = "/srv/tftp/ubuntu"

```

### DHCP server
Directly to configure the OS network installation, you must enable the option `authoritative` and specify the installation file to be sent at the connection by setting it in the `filename` parameter.

Modifying the `/etc/dhcp/dhcpd.conf` file
```
nano /etc/dhcp/dhcpd.conf
```
File after changes: [dhcpd.conf](./dhcpd.conf).


### HTTP server and preseed
First - create a symlink.

```
ln -s /srv/tftp/ubuntu /var/www/ubuntu
```

Next step is to create a file to oem.seed preset.

```
mkdir -p /var/www/html/ubuntu/server/preseed
nano /var/www/html/ubuntu/server/preseed/oem.seed
```
Copy to oem.seed all from [this file](./oem.ceed).

After creating a file you need to update `pxelinux.cfg/default` (configuration file for sturtup menu). 

```
nano /srv/tftp/ubuntu/pxelinux.cfg/default
```

*add to the end of file:*
```
label AUTO Ubuntu 18.04.01 Server

kernel ubuntu-installer/amd64/linux

append url=http://192.168.0.1/ubuntu/server/preseed/oem.seed vga=normal initrd=ubuntu-installer/amd64/initrd.gz auto=true priority=critical ramdisk_size=16432 root=/dev/rd/0 rw -

# load system from disc
label Boot from first hard disk

localboot 0x80
```

### Restart services
And the final step is to restart/start all modified services. 

```
/etc/init.d/networking restart
/etc/init.d/atftpd restart
/etc/init.d/tftpd-hpa restart
/etc/init.d/isc-dhcp-server restart
```