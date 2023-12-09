# The steps that need to be run before running `kubeadm` (or whatever I end up using).

## Install some needed things:

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'apt update && apt install -y iptables curl ca-certificates gpg jq ipvsadm' ; done
```

## kernel modules:

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF'
done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'modprobe overlay ; modprobe br_netfilter' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'lsmod | grep br_netfilter ; lsmod | grep overlay' ; done
```

## enable forwarding and bridging:

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF'
done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'sysctl --system' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward' ; done
```

## containderd and runc

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'install -m 0755 -d /etc/apt/keyrings' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg ; chmod a+r /etc/apt/keyrings/docker.gpg' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" > /etc/apt/sources.list.d/docker.list ; apt update' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'apt update && apt install -y containerd.io' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'containerd config default > /etc/containerd/config.toml' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} "sed --in-place 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml" ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'systemctl restart containerd' ; done
```

## Set up kubernetes repo:

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg' ; done
```

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" > /etc/apt/sources.list.d/kubernetes.list ; apt update' ; done
```

When the version changes from 1.18 to the next version, [look at this page](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)

```bash
for pi in {1..8} ; do ssh -t 192.168.20.5${pi} 'apt update ; apt-get install -y kubelet kubeadm kubectl ; apt-mark hold kubelet kubeadm kubectl' ; done
```
