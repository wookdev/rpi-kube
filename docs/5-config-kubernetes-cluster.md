# Configure Cluster

Things that aren't strictly needed for a k8s cluster, but that we will need.

## nginx-ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```

## democratic-csi 

Container storage interface.

We'll use the `truenas-api-*` experimental drivers so we don't have to configure
a ssh nas login with full sudo.

### Get the repo:

```bash
helm repo add democratic-csi https://democratic-csi.github.io/charts/
helm repo update
```

See what's in the repo:

```bash
helm search repo democratic-csi/
```

### Configure Storage Drivers

First get the example helm values files, and edit them.  This process combines two files
for each driver.  One file is the configuration of the storage class.  The second
is the configuration for the driver.  The two need to be combined into one file to
be applied, and the resulting file needs to be correct yaml.

### NFS Config:

Getting the two files:

```bash
wget https://raw.githubusercontent.com/democratic-csi/charts/master/stable/democratic-csi/examples/freenas-nfs.yaml -O - | sed '/INLINE/,$d' > nfs.yaml
wget https://raw.githubusercontent.com/democratic-csi/democratic-csi/master/examples/freenas-api-nfs.yaml -O - | sed -e 's/^/    /g' >> nfs.yaml
```

I change the name, nfsvers, server names, and the zfs dataset name info.  

The example `detachedSnapshotsDataSetParentName` parameter is overlapped with the
`datasetParentName` above it, right after telling you not to do that in the comment 
above it.  The two directories appear to be need to be siblings of each other at
the very least.  When I did it like the example did, the controller pod wouldn't
start up, remaining in it's crashLoopFallback status.

My file: [nfs.yaml](helm-values-files/nfs.yaml)

### iSCSI Config:

Same deal with above.  The two files are:

```bash
wget https://raw.githubusercontent.com/democratic-csi/charts/master/stable/democratic-csi/examples/freenas-iscsi.yaml -O - | sed '/INLINE/,$d' > iscsi.yaml
wget https://raw.githubusercontent.com/democratic-csi/democratic-csi/master/examples/freenas-api-iscsi.yaml -O - | sed -e 's/^/    /g' >> iscsi.yaml
```

My file: [iscsi.yaml](helm-values-files/iscsi.yaml)

The iSCSI driver needs to be privileged.  Make the namespace privileged:

```bash
kubectl label --overwrite namespace democratic-csi pod-security.kubernetes.io/enforce=privileged
```

### Install the charts:

```bash
helm upgrade --install --create-namespace --values iscsi.yaml --namespace democratic-csi zfs-iscsi democratic-csi/democratic-csi
helm upgrade --install --create-namespace --values nfs.yaml --namespace democratic-csi zfs-nfs democratic-csi/democratic-csi
```

# References:

[https://github.com/fenio/k8s-truenas](https://github.com/fenio/k8s-truenas)

[https://github.com/democratic-csi/democratic-csi](https://github.com/democratic-csi/democratic-csi)
