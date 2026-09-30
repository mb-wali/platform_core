# Harbor
.
.



# CIS – Rook Harbor Storage Migration

This procedure migrates the Harbor storage volumes to new PVCs using temporary PVCs as an intermediate storage location.

### Migration approach

1. Stop all Harbor workloads.
2. Create temporary PVCs on the new storage.
3. Create a migration pod with both the old and temporary PVCs mounted.
4. Copy the data from the old PVCs to the temporary PVCs using `rsync`.
5. Verify that the copied data is complete.
6. Delete the old Harbor PVCs/PVs.
7. Deploy/redeploy Harbor with Helm so that new PVCs are created.
8. Configure Harbor to use the newly created PVCs.
9. Copy the data from the temporary PVCs into the newly created Harbor PVCs.
10. Verify Harbor and the migrated data.
11. Remove the temporary PVCs after successful migration.

> **Important:** Do not delete the temporary PVCs until Harbor has been successfully redeployed and the data has been copied to the final PVCs.

---

## 1. Stop Harbor

Scale down all Harbor Deployments and StatefulSets:

```bash
kubectl scale deployment -n harbor --all --replicas=0

kubectl scale statefulset -n harbor --all --replicas=0
```

Verify that no Harbor application pods are still running:

```bash
kubectl get pods -n harbor
```

The migration pod will be created later and is expected to be running.

---

## 2. Create Temporary PVCs

Create temporary PVCs using the target storage class:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: migration-harbor-registry
  namespace: harbor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 150Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: migration-harbor-trivy
  namespace: harbor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 20Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: migration-harbor-database
  namespace: harbor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: migration-harbor-redis
  namespace: harbor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: migration-harbor-jobservice
  namespace: harbor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f migration-pvcs.yaml
```

Verify:

```bash
kubectl get pvc -n harbor
```

All five migration PVCs should be `Bound` before continuing.

---

## 3. Create the Migration Pod

The migration pod mounts both the existing Harbor PVCs and the temporary PVCs.

The `alpine` container installs `rsync` and then stays alive for the migration.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: harbor-storage-migration
  namespace: harbor
spec:
  restartPolicy: Never

  containers:
    - name: migration
      image: alpine:3.22
      command:
        - /bin/sh
        - -c
        - |
          apk add --no-cache rsync
          echo "Migration pod ready."
          sleep 86400

      volumeMounts:

        # OLD PVCs
        - name: old-registry
          mountPath: /old/registry

        - name: old-trivy
          mountPath: /old/trivy

        - name: old-database
          mountPath: /old/database

        - name: old-redis
          mountPath: /old/redis

        - name: old-jobservice
          mountPath: /old/jobservice

        # TEMPORARY PVCs
        - name: new-registry
          mountPath: /new/registry

        - name: new-trivy
          mountPath: /new/trivy

        - name: new-database
          mountPath: /new/database

        - name: new-redis
          mountPath: /new/redis

        - name: new-jobservice
          mountPath: /new/jobservice

  volumes:

    # OLD PVCs
    - name: old-registry
      persistentVolumeClaim:
        claimName: harbor-registry

    - name: old-trivy
      persistentVolumeClaim:
        claimName: data-harbor-trivy-0

    - name: old-database
      persistentVolumeClaim:
        claimName: database-data-harbor-database-0

    - name: old-redis
      persistentVolumeClaim:
        claimName: data-harbor-redis-0

    - name: old-jobservice
      persistentVolumeClaim:
        claimName: harbor-jobservice

    # TEMPORARY PVCs
    - name: new-registry
      persistentVolumeClaim:
        claimName: migration-harbor-registry

    - name: new-trivy
      persistentVolumeClaim:
        claimName: migration-harbor-trivy

    - name: new-database
      persistentVolumeClaim:
        claimName: migration-harbor-database

    - name: new-redis
      persistentVolumeClaim:
        claimName: migration-harbor-redis

    - name: new-jobservice
      persistentVolumeClaim:
        claimName: migration-harbor-jobservice
```

Apply:

```bash
kubectl apply -f migration-pod.yaml
```

Wait until the pod is ready:

```bash
kubectl get pod -n harbor harbor-storage-migration
```

---

## 4. Verify the Mounts

Check all mounted filesystems:

```bash
kubectl exec -n harbor harbor-storage-migration -- df -h
```

Check the data sizes on the old PVCs:

```bash
kubectl exec -n harbor harbor-storage-migration -- sh -c '
echo "=== OLD STORAGE ==="

echo "Registry:"
du -sh /old/registry

echo "Trivy:"
du -sh /old/trivy

echo "Database:"
du -sh /old/database

echo "Redis:"
du -sh /old/redis

echo "Jobservice:"
du -sh /old/jobservice
'
```

---

## 5. Copy Data to the Temporary PVCs

Open a shell in the migration pod:

```bash
kubectl exec -it -n harbor harbor-storage-migration -- sh
```

Run the migrations one by one.

### Trivy

```bash
rsync -aHAX --info=progress2 /old/trivy/ /new/trivy/
```

### Database

```bash
rsync -aHAX --info=progress2 /old/database/ /new/database/
```

### Redis

```bash
rsync -aHAX --info=progress2 /old/redis/ /new/redis/
```

### Jobservice

```bash
rsync -aHAX --info=progress2 /old/jobservice/ /new/jobservice/
```

### Registry

```bash
rsync -aHAX --info=progress2 /old/registry/ /new/registry/
```

> If an `rsync` operation is interrupted, it can be run again. Existing unchanged files will generally be skipped and only missing/changed data will be transferred.

---

## 6. Verify the Migration

Compare the old and temporary storage:

```bash
kubectl exec -n harbor harbor-storage-migration -- sh -c '

echo "=== REGISTRY ==="
du -sh /old/registry /new/registry

echo
echo "=== TRIVY ==="
du -sh /old/trivy /new/trivy

echo
echo "=== DATABASE ==="
du -sh /old/database /new/database

echo
echo "=== REDIS ==="
du -sh /old/redis /new/redis

echo
echo "=== JOBSERVICE ==="
du -sh /old/jobservice /new/jobservice

'
```

For an additional verification, compare file counts:

```bash
kubectl exec -n harbor harbor-storage-migration -- sh -c '

echo "=== FILE COUNTS ==="

echo "Registry:"
find /old/registry -type f | wc -l
find /new/registry -type f | wc -l

echo "Trivy:"
find /old/trivy -type f | wc -l
find /new/trivy -type f | wc -l

echo "Database:"
find /old/database -type f | wc -l
find /new/database -type f | wc -l

echo "Redis:"
find /old/redis -type f | wc -l
find /new/redis -type f | wc -l

echo "Jobservice:"
find /old/jobservice -type f | wc -l
find /new/jobservice -type f | wc -l

'
```

Do **not** continue until the temporary copies have been verified.

---

# 7. Delete the Old Harbor PVCs

Once the data has been successfully copied to the temporary PVCs, delete the old Harbor PVCs:

```bash
kubectl delete pvc -n harbor \
  harbor-registry \
  data-harbor-trivy-0 \
  database-data-harbor-database-0 \
  data-harbor-redis-0 \
  harbor-jobservice
```

Check:

```bash
kubectl get pvc -n harbor
```

The `migration-harbor-*` PVCs must remain.

---

# 8. Remove the Old PVs

Because the `ceph-block` StorageClass uses `Retain`, the old PVs may remain in `Released` state.

Check:

```bash
kubectl get pv
```

Identify the old Harbor PVs and delete them:

```bash
kubectl delete pv <OLD-PV-NAME>
```

> **Important:** deleting a `Retain` PV removes the Kubernetes PV object but does not automatically remove the underlying Ceph RBD image.

Do not manually delete the Ceph RBD images until the Harbor migration has been fully verified.

---

# 9. Configure Harbor to Use the Migrated PVCs
The migrated PVCs will now become the permanent Harbor PVCs.

Configure the Harbor Helm values to use the existing PVCs with existingClaim.


example:
```yaml
database:
  existingClaim: migration-harbor-database

redis:
  existingClaim: migration-harbor-redis

jobservice:
  existingClaim: migration-harbor-jobservice

registry:
  existingClaim: migration-harbor-registry

trivy:
  existingClaim: migration-harbor-trivy
```

# 10. Deploy Harbor
Apply the updated Helm/Argo CD configuration.
