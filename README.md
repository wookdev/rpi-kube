# RPi-Kube

Project to install Kubernetes on my 8 node Raspberry Pi cluster.

## Part is Parts:

The directories in this repo:

### ansible

Ansible playbooks to do some of the documented tasks.

The goal is to do the whole built in ansible, but for getting-it-up-and-running reasons only
some of the lowest hanging pieces were done via ansible.

### docs

Instructions for the installation.  For the most part, I just cut and past commands to the 
shell to do the build.

### helm-values

Decmocratic-csi is installed via helm charts.  These are the values files used.

More values files as I need them.

### kube-yaml

Files to test pvc creation once democratic-csi is installed

