---
layout: post
title:  "Time Machine Server"
date:   2023-02-04 16:30:00 -0000
categories: linux mac
---
Apple Time Machine is a backup feature in macOS. [Apple's support article](https://support.apple.com/en-us/HT201250) shows you how to backup to a USB drive. This post shows you how to setup a Time Machine server on an Ubuntu Server and use it as a network drive for backups. This is useful if you have multiple Macs and want to backup to a central location.

## Install essential packages

```bash
sudo apt-get install samba samba-common-bin avahi-daemon
```

## Configure Samba

Samba's config is located at `/etc/samba/smb.conf`. Open it in your favorite editor and replace it with these lines.

```bash
#======================= Global Settings =======================

[global]
# Fruit global config
  fruit:aapl = yes
  fruit:nfs_aces = no
  fruit:copyfile = no
  fruit:model = MacSamba

# Permissions on new files and directories are inherited from parent directory
   inherit permissions = yes

# Change this to the workgroup/NT-domain name your Samba server will part of
   workgroup = WORKGROUP

# Samba will automatically "register" the presence of its server to the rest of the network using mDNS. Since we are using avahi for this we can disable mdns registration.
   multicast dns register = no

# Server string is the equivalent of the NT Description field
   server string = %h server (Samba, Ubuntu)

# Protocol versions
  client max protocol = default
  client min protocol = SMB2_02
  server max protocol = SMB3
  server min protocol = SMB2_02

# This tells Samba to use a separate log file for each machine that connects
   log file = /var/log/samba/log.%m

# Cap the size of the individual log files (in KiB).
   max log size = 1000

# We want Samba to only log to /var/log/samba/log.{smbd,nmbd}.
# Append syslog@1 if you want important messages to be sent to syslog too.
   logging = file

# Do something sensible when Samba crashes: mail the admin a backtrace
   panic action = /usr/share/samba/panic-action %d

#======================= Share Definitions =======================

[milkshake]
  # Load in modules (order is critical!)
  vfs objects = catia fruit streams_xattr
  fruit:time machine = yes
  fruit:time machine max size = 512G
  comment = Time Machine Backup
  path = /mnt/milkshake
  available = yes
  valid users = pi
  browseable = yes
  guest ok = no
  writable = yes
```

> Note: I usually add other backup disks that I can `smb://` into and copy files to. I also add a `public` share that is accessible by everyone. So I've not disabled mDNS registration (I've removed `multicast dns register = no` from the conf).

## Configure Avahi

Now network computer discovery is a bit separate from the actual file-sharing connection. To make the network drive visible in the network the avahi-daemon needs to installed on the server.

Create the following avahi service file

```bash
sudo vi /etc/avahi/services/samba.service
```

and add the following lines

```bash
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
  <name replace-wildcards="yes">%h</name>
  <service>
    <type>_smb._tcp</type>
    <port>445</port>
  </service>
  <service>
    <type>_device-info._tcp</type>
    <port>0</port>
    <txt-record>model=TimeCapsule8,119</txt-record>
  </service>
  <service>
    <type>_adisk._tcp</type>
    <txt-record>dk0=adVN=milkshake,adVF=0x82</txt-record>
    <txt-record>sys=waMa=0,adVF=0x100</txt-record>
  </service>
</service-group>
```

## Enable and start services

```bash
sudo systemctl enable smbd avahi-daemon
sudo systemctl start smbd avahi-daemon
```

Check status or restart services

```bash
sudo systemctl status smbd
sudo systemctl status avahi-daemon

sudo systemctl restart smbd avahi-daemon
```

## Credits and References

- [Setup Apple Time Machine network drive with Samba on Ubuntu 22.04](https://blog.jhnr.ch/2023/01/09/setup-apple-time-machine-network-drive-with-samba-on-ubuntu-22.04/)
- [Samba as a file server](https://ubuntu.com/server/docs/samba-file-server)