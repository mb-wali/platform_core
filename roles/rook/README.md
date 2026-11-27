# Rook

Ceph cluster running on Kubernetes.

---

## Prerequisites

Each Kubernetes node participating in the Ceph cluster must have an additional dedicated disk available for Ceph storage.

### Disk Requirements

For each Ceph storage disk:

- **Minimum capacity:** 100 GB (for now)
- **Storage type:** SSD-backed storage; **NVMe preferred**
- **Dedicated disk:** The disk must be a separate block device and must not be part of the OS disk
- **Raw disk:** The disk must **not** be partitioned, formatted, or mounted
- **No existing data:** The disk must be empty and contain no data that needs to be preserved
- **Available to Rook:** The entire raw block device must be available for Rook/Ceph to use

### Example

The node should have a layout similar to:

```text
/dev/sda    → OS disk
/dev/sdb    → Dedicated Ceph disk (100 GB+, SSD/NVMe)
```

---

## Ceph Dashboard Access

The Rook-Ceph dashboard uses the following default credentials:

- **Username:** `admin`
- **Password:** Stored in the Kubernetes Secret `rook-ceph-dashboard-password`

Retrieve the dashboard password with:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 --decode
```
