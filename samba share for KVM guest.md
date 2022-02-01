# Samba Folder Share for a KVM Guest

## Context

I had surprisingly a lot of trouble setting this up. I'm using OpenSUSE Tumblewheel, and the below configuration ensures that a Host samba share is accessible from a kvm (QEMU) guest when you have a firewall (firewalld) in place. A combination of different resources made this work, therefore I wanted to consolidate the steps in a single file. This being Github, please feel free to comment all for the benefit of the open source community.

Setting up a (passthrough) folder share on the Host O/S available to the guest O/S only

- Your HOST O/S is a Linux-based distro (Ubuntu, Arch, RHEL, etc.)
- You have the following installed: qemu, libvirt (Virtual Machine Manager), bridge-utils, virtio
- You're using Virtual Machine Manager (libvirt) + QEMU to run a Guest O/S
- Your Guest O/S can be either Linux based or Windows

This tutorial was done using OpenSUSE Tumblewheel.
Ensure to follow your distribution's package manager instructions to install Samba

## Verify Samba on Host O/S

### Verify that Samba is running

```
systemtl status smb nmb
```

### Samba Configuration

Config file location: ```/etc/samba/smb.conf```

#### Global Configuration

Configurations with a comment are those I focus on to make this tutorial happen

```
[global]
    security = user                          # User-level authentication using a server's account
    workgroup = WORKGROUP                    # default value is WORKGROUP, but you can change the WORKGROUP here. If you change the value, note below where WORKGROUP is referred to and replace by the value set here
    passdb backend = tdbsam
    ntlm auth = Yes                          # For Windows guests specifically, allows NTLM v1 + negotiation
    map to guest = Bad User                  # The failure to login assigns the user a Guest account
    logon path = \\%L\profiles\.msprofile
    logon home = \\%L\%U\.9xprofile
    logon drive = P: 
    hosts allow = 192.168.100.0/24           # Limits Samba to a specific range of IP addresses. This ip range encompass the IP set to your bridge NIC (explained below)
```

#### Share-specific Configuration

```
[win10]
    path = /home/kev/qemu-mounts/Win10-Mount
    browseable = Yes
    guest ok = No                            # Not allow guest accounts on the share
    valid users = kev                        # Only allow the list of users
    read only = No                           # Allow read-write
    create mask = 0755                       # Permissions for file created by Samba
```

#### Associate one of your unix users to Samba

replace user by the linux user (whether it's a new linux user, your current user, etc.)

```shell
$ sudo smbpasswd -a user 
```

You can type a different password, or the same one.

Now that a linux user is associated to Samba, restart samba

```shell
$ sudo systemctl restart smb nmb
```

Once Samba is up again and configured, let's validate by accessing it from the host by navigating in a folder (such as Dolphin) and use the address ```smb://your-hostname```.

## Create a bridge interface

Create a bridge network using libvirt, bridge-utils brctl, etc. and ensure to give it an ip range that matches the above hosts-allow value:

my bridge info:

```shell2
$ ip a
...
5: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:6a:37:5a brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global virbr0
       valid_lft forever preferred_lft forever
```

## Verify Firewall configurations

Once Samba is configured and has been validated on the host, verify your firewall configuration (if any).
In my case, the firewall is firewalld (also manageable via the YaST Firewall GUI)

#### Verify the firewall is running

```shell
$ systemctl status firewalld
```

#### Associate a zone to the bridge, in my case "virbr0", the default bridge created by libvirt.

```shell
$ sudo firewall-cmd --permanent --zone=libvirt --add-interface=virbr0'
```

#### Get the zone associated to the interface

```shell
$ sudo firewall-cmd --get-zone-of-interface=virbr0
libvirt

$ sudo firewall-cmd --list-all --zone=libvirt
libvirt (active)
  target: ACCEPT
  icmp-block-inversion: no
  interfaces: virbr0
  sources: 
  services: dhcp dhcpv6 dns ssh tftp
  ports: 
  protocols: icmp ipv6-icmp
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
        rule priority="32767" reject
```

#### Add Samba and samba-client to libvirt

```shell
$ sudo firewall-cmd --permanent --zone=libvirt --add-service=samba

$ sudo firewall-cmd --permanent --zone=libvirt --add-service=samba-client

$ sudo firewall-cmd --list-all --zone=libvirt
libvirt (active)
  target: ACCEPT
  icmp-block-inversion: no
  interfaces: virbr0
  sources: 
  services: dhcp dhcpv6 dns samba samba-client ssh tftp
  ports: 
  protocols: icmp ipv6-icmp
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
        rule priority="32767" reject
```

## Testing Samba from the Guest O/S

At this point, we have confirmed that the Samba share is accessible from the host, and that the related ports are open on the libvirt zone, the zone related to the bridge interface that will be used as the NIC of the Guest O/S. Now, all that's left is to create your instance of Linux or Windows Guest O/S using KVM:

- Use the Bridge interface as your NIC
- Once booted, access to the share
  - Linux: ```smbclient -L HOST_IP_ADDRESS -U "WORKGROUP\user"```
  - Windows (in a cmd): ```net use \\HOST_IP_ADDRESS /USER:"WORKGROUP\user"```