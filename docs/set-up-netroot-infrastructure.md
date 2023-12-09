# Infrastructure

The Raspberry Pi 4 boots like this when it boots over the network (as I've configured it):

1. The rpi eeprom is configured with a tftp server IP, and a format for identifying itself.
2. On startup it gets an IP address via DHCP.
3. Downloads files for the second-stage bootloader from the tftp server
4. The second stage bootloader initializes the hardware, and loads the kernel and initrd
   image, then turns control over to them.
5. The kernel mounts the root volume and finishes booting

We need, thusly:

1. TFTP storage, with subdirectories for each rpi by mac address
2. TFTP storage is also shared via NFS
3. Linux root volumes, with subdirectories for each rpi by ip address
4. Each rpi needs a zvol for the block device shares
5. iSCSI configurations one zvol for each pi root volume
6. Access to the main share for storing stuff, called "space"

## NAS Storage Plan

It looks like this:

```
/mnt/z1
|
+-- k8s (datastore)
|   |
|   +-- iscsi (datastore)
|   +-- iscsisnaps (datastore)
|   +-- nfs (datastore)
|   +-- nfssnaps (datastore)
|
+-- rpi (datastore)
|   |
|   +-- pi-1-zvol (zvol, iscsi)
|   +-- pi-2-zvol (zvol, iscsi)
|   +-- etc...
|   +-- pi-8-zvol (zvol, iscsi)
|
+-- tftp (datastore, tftp, nfs)
|   |
|   +-- dc-a6-32-e9-00-4c (dir)
|   +-- dc-a6-32-e9-00-60 (dir)
|   +-- etc...
|
+-- space
```

I set them all up with default configs.

### `/mnt/z1/k8s`

This is where the container storage interface will create nfs and iscsi volumes.

Versions of TrueNAS prior to 13 had a limit on dataset names of (I think) 63 or 64
characters.  Democratic CSI and/or Kubernetes creates volume names that are like
47 characters long, so the location prefixes had to be short.  Not a problem now,
but I've structured my whole storage plan around this because I started this
project when TrueNAS 12 was the thing.

### `/mnt/z1/rpi`

Store the zvols that are the iSCSI extents for the root volumes.  Not shown above,
but this was also the locations of the NFS root volumes, and those datasets still
exist just in case, and might show up in screenshots.

### `/mnt/z1/tftp`

The directories in here are the RPi mac addresses used by the RPi bootloader to
distinguish themselves from each other.  Mac addresses are usually specified colon-delimited
(dc:a6:32:e9:00:4c) but the colon is a dangerous character for filenames on some
systems[^1].  The directories the bootloader wants are dash-delimited instead, which
can cause a problem if you fail to do that conversion before you code them.

[^1]: I'm looking at you Classic Macs.

The NFS export is a convenient way of getting files into the tftp directories
in the first place and then updating them on OS updates after that.  We only need
the export of the tftp directory itself because we can mount the subdirectories
directly.


## NFS

NFS is configured with NFSv4 turned on.

The NFS shares follow the "top level" datasets:

```
/mnt/z1/rpi -- not shown above
/mnt/z1/tftp
```

Both shares are configured with `maproot user` and `maproot group` set to `root` user
and group.  I also add the following two networks to all NFS shares in my system:

```
192.168.10.0/24
192.168.20.0/24
```

Just because.

## TFTP

I'm using the tftp service in SCALE, even tho it is deprecated in favor of the
tftp-hpa application.

The app has some issues:

1. The UI will not allow you to specify tftp and nfs shares on the same dataset
2. The app mangles the permissions of the dataset you choose for storage so they
   can't be access by anything else

The Raspberry Pis boot via tftp, but once running updates to that directory have
to be made over nfs.

The tftp service was configured with all defaults with storage set to `/mnt/z1/tftp`.

## iSCSI

We need a portal to talk to, initiator groups to allow access.  The portal is
on the interface closest to the RPi systems.  The initiator group allows all
initiators.

![iscsi portal definition](pics/iscsi-portal.png)

We set up iscsi targets and associated froo-fra only for the RPi root volumes.
K8s will take care of creating all of its own iscsi shares given the portal and
initiator group numbers.

The mapping of the root volumes is obvious and simple:

| target | lun ID |  extent   | zvol      |
|:-------|:------:|:---------:|-----------|
| pi-1   |   0    | pi-1-root | pi-1-zvol |
| pi-2   |   0    | pi-2-root | pi-2-zvol |
| etc... |        |           |           | 

The targets are defined like this:

![target pi-1 definition](pics/iscsi-target.png)

The extents are defined like this:

![extent pi-1 definition](pics/iscsi-extent.png)

## SSH

This configuration is on the Mac or whatever you use to access these systems.

Because host keys are going to change back and forth all the time during this
process, I added the following stanza to my `~/.ssh/config` file:

``` config
Host pi-* 192.168.20.5?
 User root
 StrictHostKeyChecking no
 UserKnownHostsFile=/dev/null
```

It doesn't check host keys, which is okay because `/dev/null` is generally empty,
and it doesn't add anything to the `known_hosts` file even tho it says it is
because `/dev/null` is generally empty.
