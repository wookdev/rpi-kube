# But What Happened Was...

The list of things I wanted to do at the beginning vs. what actually happened, and why:

## NFS roots

Originally I set the Pis up to have NFS roots.  In our environment, there were two problems[^1]
that made this really hard.  And then, once it was working it turned out NFS was a bad choice
because some things don't like being on NFS.  Python, containerd.  I know Java doesn't do
well on it either from past experience.  This has to do with capabilities of real filesytstems
vs the sort of half-baked model of NFS.

I went back to iSCSI roots, which I'd done before and was pretty comfortable with.
The notes for NFS root builds are included.

The two main exports are at that spot so they can be mounted to create all the
subdirectories needed under them without having to log into the nas via ssh, or
create individual datastores.

The RPi will mostly be mounting subtree directories of the two main exports.
This is apparently assumed in NFSv4.  NFSv3 has an option on the export to enable
or disable subtree directory mounting.  TrueNAS SCALE sets up all nfs exports
with the `no_subtree_check` option.  It does not have an "All Dirs" option like
CORE did.  This prevents clients from mounting directories below the exported
directory.

This shouldn't be a problem.  But it turned out to be a problem because there's a bug
in Debian since 2007 where the `nfsmount` program copied into the initrd.img doesn't
do nfs v4.  It turns out there's a workaround, by adding a hook that copies `mount.nfs`
into the initrd.img during `update-initramfs`. 

That workaround worked perfectly.  I just want the week of my life (Fri to Thurs)
I spent working on it back.

## k0s cluster installer

In a previous run thru a few years back I ended up deploying my cluster with `kubeadm` and was
confused by it.  Of all the assorted k8s installers I kind of latched onto the [K0s project]().

It accomplishes its goal by including **EVERYTHING** in one huge binary.  This made my skin
crawl for no reason I can explain.  It also means typing a lot of commands starting with
four extra keystrokes ('k0s kubectl...'), which annoys the grade-A-lazy-typist in me.

Over time my brain wrapped itself around `kubeadm` and it didn't seem so confusing this time
around so that's what I went with.

## Alpine Linux and CoreOS

Alpine LInux is a linux distro that is so small it almost isn't a distro.  Teeny, with just
enough OS to get applications running.  I became convinced it couldn't boot from the network.
So bailed on it.

CoreOS is small too.  But its images all boot using u_boot[^2], and I didn't want to unwind it
to do the boot in "the RPi way".

I don't like Ubuntu.  Especially the RPi images.  They contain everything Canonical and
Ubuntu, needed or not, as some kind of advertisement.  While looking thru the NFS root
stuff I happened upon `debootstrap` and used it to build very minimal Ubuntu installs.

## Kube-VIP

This seemed to work well.  I had problems
restarting the control plane.  This is a Raspberry Pi cluster, it's going to be stopped
and started every now and again.  Event when configured as a static pod kube-vip doesn't
run until k8s itself is running.

Decided to just run with one control plane node, and leave it at that.  If I bork the
cluster somehow, I can just restore the nodes to snapshots and re-create it with either
ansible and these docs.

[^1]: One is this debian bug from 2007 where `nfsmount` that is included
in `initrd.img` can't do NFSv4.

[^2]: One day I'll spend some cycles on learning u_boot, and then probably hate myself for
not doing so sooner.

[k0s project]: https://k0sproject.io/
[Canonical]: https://canonical.com/
[Ubuntu]: https://ubuntu.com/
