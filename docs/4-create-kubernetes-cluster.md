# Installing actual Kubernetes things and making a cluster


## Configure the cluster with `kubeadm`:

### On `pi-1.local`:

Initialize the cluster.  We're just doing a single node control plane.

```bash
kubeadm init --control-plane-endpoint=cluster.local --pod-network-cidr=10.0.0.0/8 --upload-certs
```

**OMG COPY THE JOIN INSTRUCTIONS**

Install kube-router:

```bash
KUBECONFIG=/etc/kubernetes/admin.conf kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter-all-features.yaml
```

Remove kube-proxy:

```bash
KUBECONFIG=/etc/kubernetes/admin.conf kubectl -n kube-system delete ds kube-proxy
```

Clean up kube-proxy:

```bash
ctr images pull registry.k8s.io/kube-proxy:v1.28.3
ctr run --rm --privileged --net-host --mount type=bind,src=/lib/modules,dst=/lib/modules,options=rbind:ro registry.k8s.io/kube-proxy:v1.28.3 kube-proxy-cleanup kube-proxy --cleanup
```

### On pi-2.local thru pi-8.local

Also do exactly what the `kubeadm init` told us to do on the worker nodes:

```bash
kubeadm join cluster.local:6443 --token d856qk.xxzc00vc638ph1lh --discovery-token-ca-cert-hash sha256:d466c025847a11bf0e3f04f194c2010ca6bb9ca9ae4cf6919b16aaab08bd13d8
```
