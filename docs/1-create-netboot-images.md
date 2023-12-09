#linux #rpi #ubuntu #iscsi

# Set Up Host System

Download the latest Ubuntu release and flash it onto a mini-SD card.  Boot the
Pi from it.

This pi is the "host pi" for now.  We'll use it for all these processes.

First, do a full upgrade and Install some things we need:

```bash
apt update && apt full-upgrade && apt install -y nfs-common open-iscsi debootstrap linux-modules-extra-raspi
```

After the upgrade, reboot so we have a current kernel running

# Mount iSCSI Root Volumes

Do an iscsi discovery.

```bash
iscsiadm -m discovery -t st -p 192.168.20.20
```

Results should look like this:

```
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-1
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-2
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-3
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-4
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-5
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-6
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-7
192.168.20.20:3260,1 iqn.1998-02.net.munged.nas1:pi-8
```

Attach to all these targets:

```bash
iscsiadm -m node -L all
```

That will attach these devices as disk devices, like `/dev/sda` and such.  It
will not do them in order.  You can instead attach them like this:

```bash
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-1 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-2 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-3 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-4 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-5 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-6 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-7 -l
iscsiadm -m node -T iqn.1998-02.net.munged.nas1:pi-8 -l
```

This will probably make the `pi-1` target be `/dev/sda` and `pi-8` be `/dev/sdh`.
But it might not either.  It isn't actually important right at this moment.

Create a partition, write a filesystem:

```bash
for dv in a b c d e f g h ; do parted /dev/sd${dv} mklabel gpt ; done
for dv in a b c d e f g h ; do parted --align optional /dev/sd${dv} mkpart primary ext4 0% 100% ; done
for dv in a b c d e f g h ; do mkfs.ext4 /dev/sd${dv}1 ; done
```

Once `mkfs.ext4` is run, the devices will have UUIDs in `blkid`.  Figure out mapping
of UUID of the new devices to which target, and thus which pi, they belong to.  I did
it sorta like this:

```bash
iscsiadm -m session -P 3
blkid
```

The `iscsiadm` command will give you info on which targets are attached to which devices.
The `blkid` command will map devices to UUIDs.  Combine these two to come up with this
for the `/etc/fstab` of the host pi system.

Capture the data from this to create [[Init build-vars.sh]].  It also needs the tftp
mac address directory names, which should exist already, but are mentioned explicitly
when creating the directories, below.

```
UUID=7779e3ee-c0dd-495e-923e-b9c2be5c0a62 /mnt/pi-1 ext4 _netdev,noauto 0 0
UUID=aeb22000-0bf7-4489-943f-2774b3bd060a /mnt/pi-2 ext4 _netdev,noauto 0 0
UUID=43b1fb7f-0f15-4aa0-9024-d3b0fe18f96e /mnt/pi-3 ext4 _netdev,noauto 0 0
UUID=24b796db-80c5-4f77-80aa-39eb71046a11 /mnt/pi-4 ext4 _netdev,noauto 0 0
UUID=25443955-794b-490b-a420-5338c2af031d /mnt/pi-5 ext4 _netdev,noauto 0 0
UUID=946184be-654c-451a-be42-58c6bda6b202 /mnt/pi-6 ext4 _netdev,noauto 0 0
UUID=df13b02e-22dd-442a-9624-b720c49020bc /mnt/pi-7 ext4 _netdev,noauto 0 0
UUID=80a0e216-1e9d-4996-b505-5736007be692 /mnt/pi-8 ext4 _netdev,noauto 0 0
```

Create mount points for the roots and then mount them:

```bash
mkdir /mnt/pi-{1..8}
for pi in /mnt/pi-* ; do mount ${pi} ; done
```

# Firmware files

Create mount point and mount the firmware (`tftp`) export:

```bash
mkdir /mnt/tftp
mount -t nfs nas1.local:/mnt/z1/tftp /mnt/tftp
```

Copy the contents of `/boot/firmware` to the boot nfs mounts. :

```bash
for mac in /mnt/tftp/dc-* ; do rsync -avhP /boot/firmware/ ${mac} ; done
```

Create env vars files: [[Init build-vars.sh]]

Edit `/mnt/tftp/config.txt`

```bash
for mac in /mnt/tftp/dc-* ; do
cat <<EOF | tee ${mac}/config.txt
[all]
kernel=vmlinuz
cmdline=cmdline.txt
initramfs initrd.img followkernel

arm_boost=1

dtparam=audio=off
dtparam=i2c_arm=on
dtparam=spi=on

enable_uart=0
ignore_lcd=1

camera_auto_detect=0
display_auto_detect=1

# Config settings specific to arm64
arm_64bit=1
dtoverlay=dwc2

# Turn off some things:
dtoverlay=disable-bt
dtoverlay=disable-wifi
EOF
done
```

Edit `/mnt/tftp/cmdline.txt`. 

```bash
for mac in /mnt/tftp/dc-*
do
source ${mac}/build-vars.sh
cat <<EOF | tee ${mac}/cmdline.txt
dwc_otg.lpm_enable=0 console=tty1 ip=dhcp rootwait fixrtc root=UUID=${PI_UUID} ISCSI_INITIATOR=iqn.1998-02.net.munged:${PI_HOST} ISCSI_TARGET_NAME=iqn.1998-02.net.munged.nas1:${PI_HOST} ISCSI_TARGET_IP=192.168.20.20 ISCSI_TARGET_PORT=3260
EOF
done
```

Optional: Clean up files not needed in this build:

```bash
rm -rf /mnt/tftp/dc-*/vmlinu* /mnt/tftp/dc-*/initrd* /mnt/tftp/dc-*/*.bak /mnt/tftp/dc-*/overlays/*.bak /mnt/tftp/dc-*/.Spotlight-V100 /mnt/tftp/dc-*/.fseventsd
```

# Create Root Image

Use `debootstrap` to create a minimal ubuntu install on the root volumes on each iscsi drive:

```bash
for pi in /mnt/pi-* ; do debootstrap jammy ${pi} http://ports.ubuntu.com/ubuntu-ports ; done
```

We need some directories for mount points inside the root volumes:

``` shell
mkdir /mnt/pi-{1..8}/boot/dtbs /mnt/pi-{1..8}/boot/firmware /mnt/pi-{1..8}/mnt/space
```

Copy the `dtbs` into image boot directories:

```bash
for pi in /mnt/pi-* ; do rsync -avhP /boot/dtb* ${pi}/boot ; done
```

Bind mount `/proc`, `/dev`, and `/sys` into the new root directory so we can run programs
in the `chroot` environment:

```bash
for pi in /mnt/pi-* ; do 
mount --bind /proc ${pi}/proc
mount --bind /dev ${pi}/dev
mount --bind /sys ${pi}/sys
done
```

And bind mount the firmware directories too:

```bash
for mac in /mnt/tftp/dc-* ; do
source ${mac}/build-vars.sh
mount --bind ${mac} /mnt/${PI_HOST}/boot/firmware
done
```

Set up correct apt sources.  The file is `/etc/apt/sources.list` on each drive:

```bash
for pi in /mnt/pi-* ; do
cat <<EOF | tee ${pi}/etc/apt/sources.list
deb http://ports.ubuntu.com/ubuntu-ports jammy main restricted
deb http://ports.ubuntu.com/ubuntu-ports jammy-updates main restricted
deb http://ports.ubuntu.com/ubuntu-ports jammy-security main restricted
EOF
done
```

Set up `fstab`:

```bash
for mac in /mnt/tftp/dc-* ; do
source ${mac}/build-vars.sh
cat <<EOF | tee /mnt/${PI_HOST}/etc/fstab
#
# nfs mounts -- root must come first
# 
UUID=${PI_UUID} / ext4 _netdev 0 0
192.168.20.20:/mnt/z1/tftp/${PI_MAC} /boot/firmware nfs4 _netdev,noatime 0 0

192.168.20.20:/mnt/z1/space /mnt/space nfs noauto,noatime 0 0
EOF
done
```

Create `/etc/kernel/postinst.d/chmod-vmlinuz` on each drive and chmod it executable:

```bash
for pi in /mnt/pi-* ; do
cat <<EOF | tee ${pi}/etc/kernel/postinst.d/chmod-vmlinuz
#!/bin/sh
chmod 0644 /boot/vmlinuz-*
EOF
chmod a+x ${pi}/etc/kernel/postinst.d/chmod-vmlinuz
done
```

We need to install a kernel, initrd, iscsi, nfs, and ssh.  Ignore errors for now:

```bash
for pi in /mnt/pi-* ; do
chroot ${pi} apt update
chroot ${pi} apt upgrade -y
chroot ${pi} apt install -y linux-raspi nfs-common openssh-server open-iscsi linux-modules-extra-raspi
done
```

Fix locales:

```bash
for pi in /mnt/pi-* ; do
    chroot ${pi} dpkg-reconfigure --frontend=noninteractive locales
    chroot ${pi} update-locale LANG=en_US.UTF-8 LC_MESSAGES=POSIX
done
```

Set iscsi config file for iscsi root volumes:

```bash
for pi in /mnt/pi-* ; do
	sed -i "s/node.conn\[0\].timeo.noop_out_interval = 5/node.conn\[0\].timeo.noop_out_interval = 0/" ${pi}/etc/iscsi/iscsid.conf
	sed -i "s/node.conn\[0\].timeo.noop_out_timeout = 5/node.conn\[0\].timeo.noop_out_timeout = 0/" ${pi}/etc/iscsi/iscsid.conf
	sed -i "s/node.session.timeo.replacement_timeout = 120/node.session.timeo.replacement_timeout = 86400/" ${pi}/etc/iscsi/iscsid.conf
done
```

Tag iscsi to be included in the initrd and then `update-initramfs`:

```bash
for pi in {1..8} ; do
cat <<EOF | tee /mnt/pi-${pi}/etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1998-02.net.munged:pi-${pi}
EOF
touch /mnt/pi-${pi}/etc/iscsi/iscsi.initramfs
chroot /mnt/pi-${pi} update-initramfs -k $(uname -r) -c
done
```

Set up iscsi discovery db, and enable sshd:

```bash
for pi in /mnt/pi-* ; do
chroot ${pi} iscsiadm -m discovery -t st -p 192.168.20.20
chroot ${pi} systemctl enable ssh
done
```

`flash-kernel` doesn't run in `chroot` environments.  Or maybe it doesn't run because it was being installed.  We must copy the kernel and initrd into the boot partition.

```bash
for mac in /mnt/tftp/dc-* ; do
source ${mac}/build-vars.sh
cp /mnt/${PI_HOST}/boot/vmlinuz ${mac}
cp /mnt/${PI_HOST}/boot/initrd.img ${mac}
done
```

Enable root logins (with keys only):

```bash
for pi in /mnt/pi-* ; do
cat <<EOF | tee ${pi}/etc/ssh/sshd_config.d/permit-root-login.conf
PermitRootLogin prohibit-password
EOF
done
```

Set up logins for root with an ssh key.  This assume we have that key set up already for the host rpi root user:

```bash
for pi in /mnt/pi-* ; do
mkdir ${pi}/root/.ssh
cp /root/.ssh/authorized_keys ${pi}/root/.ssh/
chmod 700 ${pi}/root/.ssh
chmod 600 ${pi}/root/.ssh/authorized_keys
done
```


# Reboot without sd card

In theory, the pi should boot up and offer a login.

Once logged in continue to:

[[4 - Configure Netboot Image]]







Refs:
https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfsroot.html
