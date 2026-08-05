# [Ceph-CSI](https://github.com/ceph/ceph-csi)

Ceph Container Storage Interface (CSI) driver for RBD, CephFS.


## Set RBD storageclass as default

```bash
kubectl patch storageclass ceph-rbd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

# Interact with your Ceph Cluster
**Install cli client**
```bash
sudo apt update
sudo apt install ceph-common
```

---

## CephRBD
How to get existing volumes of images

1. create your keyring file
```bash
# ceph.keyring
[client.rdmkube]
key = <user-key>
```

2. Create ceph conf
```bash
# ceph.conf
[global]
mon_host = <IP>:6789,<IP>:6789,<IP>:6789
```

3. RBD commands

```bash
# list aval volumes
rbd ls --pool=rdmkube --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf

# list with Volume usage
rbd du --pool=rdmkube --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf

# Volume info on selected volume
# rbd info <volume> --pool=rdmkube --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf
rbd info csi-vol-2859e85d-5d0b-42a0-a0da-ba9475fb2db4 --pool=rdmkube --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf


# check aval pool sizes
ceph df --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf
```

4. Find which PVC is which image in CEPH

```bash
# This will give you the exact image name which is in ceph
kubectl get pv pvc-b2a2b5ef-d8c5-494e-9815-c335498fa2c5 -o yaml |grep imageName
```

5. Deleting a cephRBC image (WARNING)
```bash
rbd rm csi-vol-d1365722-d99a-4db9-8245-a7ebc33e6bdd --pool=rdmkube --id rdmkube --keyring ./ceph.keyring --conf ./ceph.conf
```

---

## CephFS

```bash
CephFS filesystem
└── subvolume group (e.g. rdmprodcsi | /volumes/rdmprodcsi)
    └── subvolume (one per PVC)
        └── actual directory used by the pod
```

### commands

```bash
# get subvolume info
ceph fs subvolume info cephfs_kube /volumes/rdmprodcsi --id rdmprodkubefs --conf ./ceph-fs.conf --keyring ./ceph-fs.keyring


# list volumes inside subvol
ceph fs subvolume ls cephfs_kube /volumes/rdmprodcsi --id rdmprodkubefs --conf ./ceph-fs.conf --keyring ./ceph-fs.keyring
## or 
ceph fs subvolume ls cephfs_kube --group_name rdmprodcsi --id rdmprodkubefs --conf ./ceph-fs.conf --keyring ./ceph-fs.keyring


# Check usage for one PVC - volume inside subvolume group
ceph fs subvolume info cephfs_kube csi-vol-6908b411-4cad-4c5b-98a3-2f511fc68a5f --group_name rdmprodcsi --id rdmprodkubefs --conf ./ceph-fs.conf --keyring ./ceph-fs.keyring


# delete this subvolume inside the subvolume group
ceph fs subvolume rm cephfs_kube csi-vol-6908b411-4cad-4c5b-98a3-2f511fc68a5f --group_name rdmprodcsi --id rdmprodkubefs --conf ./ceph-fs.conf --keyring ./ceph-fs.keyring
### error deleting: Error EXDEV: error in rename /volumes/rdmprodcsi/csi-vol-6908b411-4cad-4c5b-98a3-2f511fc68a5f to /volumes/_deleting/60f779ac-d632-4e99-8fc7-04e27294e368
```

# Issues
* CephFS v3.16 : [ceph.dir.subvolume](https://www.osso.nl/blog/2023/cephfs-einval-specified-for-ceph-dir-subvolume/)

