# rpi-kube-docs

Notes on creating a k8s cluster on raspberry pis.  The code in the source
blocks is what I do -- copy-and-paste -- to create the nodes.  I also turned some
of it into Ansible roles and playbooks.

This process starts at nothing on the RPis at all thru having a working k8s cluster.[^1]

## Involved Entities:

* 8 Raspberry Pi 4B 8Gig computers
  - `pi-1.local` thru `pi-8.local`
  - They all have RPi PoE+ HATs on them for power

* 1 TrueNAS SCALE 22.12.4.2
  - `nas1.local`
  - 10Gbe connection to switch


## Configuration Decisions

At a high-level these were the deployment goals:

* Network booting (no SD cards)
* tftp/nfs boot volume
* iSCSI root volume
* minimal Ubuntu image
* k8s cluster bootstrap `kubeadm`
* kube-router for a cni
* single control plane node
* democratic-csi for storage
* cert-manager for managing certs


## Pages and Processes:

1. [Create the netboot Ubuntu images]() -- Builds the firmware and root filesystems
2. [Configure the netboot images]() -- Boot from images and configure them
3. [Prep images for the K8s installs}]() -- Load repos and assorted settings
4. [Bootstrap the K8s cluster]() -- Create the control plane, apply the CNI, and join workers
5. [Configure the K8s cluster]() -- Configure the CSI

And...


## Sites that Helped Me

These two helped me understand iscsi and using them as root volumes for the RPis.

* [Shawn Wilsher]() -- Network Booting a Raspberry Pi 4 with an iSCSI Root via FreeNAS
* [Matt Olan]() -- Raspberry Pi iSCSI Root on Ubuntu 20.04


## Sideline stuff:

* Sensu for observability and basic monitoring

[^1]: That promise is from another version of these notes that may or may not have been
converted to this repo when you read this.  I intend to import all of them.

[Create the netboot Ubuntu images]: 1-create-netboot-images.md
[Configure the netboot images]: 2-configure-netboot-image.md
[Prep images for the K8s installs}]: 3-prep-kubernetes.md
[Bootstrap the K8s cluster]: 4-create-kubernetes-cluster.md
[Configure the K8s cluster]: 5-config-kubernetes-cluster.md

[Shawn Wilsher]: https://shawnwilsher.com/2020/05/network-booting-a-raspberry-pi-4-with-an-iscsi-root-via-freenas/
[Matt Olan]: https://matt.olan.me/post/raspberry-pi-iscsi-root-on-ubuntu-20-04/
