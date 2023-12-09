Now that we've set up the image, boot from it.

SSH in as root.

Install some useful tools:

``` bash
apt update && apt install -y vim rpi-eeprom libraspberrypi-bin curl file
```

Hostname:

```
hostnamectl set-hostname pi-1.local
```

Time zone:

```
timedatectl set-timezone America/New_York
```

Recreate ssh host keys to make sure they are unique to each host then restart ssh:

``` bash
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
systemctl restart ssh.service
```

Recreate machine-id files to make sure they are unique to each host:

``` bash
rm -fv /etc/machine-id
rm  -v /var/lib/dbus/machine-id
dbus-uuidgen --ensure=/etc/machine-id
dbus-uuidgen --ensure
reboot
```

Minor cleanup:

``` bash
apt -y autoremove && apt autoclean
```

At this point, the systems are ready for whatever may be needed.