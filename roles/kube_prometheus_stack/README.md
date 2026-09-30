# [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)


---

# Grafana PVC Migration: `CSI` → `ROOK`

## Description

This procedure migrates the Grafana persistent volume used by `kube-prometheus-stack` from the old `ceph-rbd` StorageClass to the new `ceph-block` StorageClass.

This migration is done using a temporary PVC because the current Helm configuration does not support using an existing PVC for Grafana in the required way.



1. Stop Grafana

Stop Grafana and the related kube-prometheus-stack components:

```bash
kubectl scale --replicas=0 -n monitoring \
  deployment.apps/kube-prometheus-stack-grafana \
  deployment.apps/kube-prometheus-stack-kube-state-metrics \
  deployment.apps/kube-prometheus-stack-operator
```

2. Create Temporary PVC

Create `temp-pvc.yaml`:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-migration
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ceph-block
  volumeMode: Filesystem
```

Apply:

```bash
kubectl apply -n monitoring -f temp-pvc.yaml
```



3. Copy Old PVC → Temporary PVC
Create `migration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grafana-volume-migration
  namespace: monitoring
spec:
  restartPolicy: Never
  containers:
    - name: migration
      image: alpine:3.20
      command:
        - /bin/sh
        - -c
        - |
          set -eu
          echo "OLD:"
          du -sh /old
          echo "NEW:"
          du -sh /new
          echo "Copying..."
          cp -a /old/. /new/
          echo "DONE"
          echo "OLD:"
          du -sh /old
          echo "NEW:"
          du -sh /new
      volumeMounts:
        - name: old
          mountPath: /old
          readOnly: true
        - name: new
          mountPath: /new
  volumes:
    - name: old
      persistentVolumeClaim:
        claimName: kube-prometheus-stack-grafana
    - name: new
      persistentVolumeClaim:
        claimName: grafana-migration
```

Apply:

```bash
kubectl -n monitoring apply -f migration.yaml
```

4. Delete the Old Grafana PVC
Once the data has been successfully copied to `grafana-migration`, delete the old Grafana PVC:

```bash
kubectl delete pvc kube-prometheus-stack-grafana -n monitoring
```

5. Redeploy grafana to create new PVC using new Storageclass

6. Copy Temporary PVC → New Grafana PVC
Create `copyfromtemp.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grafana-volume-migration-2
  namespace: monitoring
spec:
  restartPolicy: Never

  containers:
    - name: migration
      image: alpine:3.20
      command:
        - /bin/sh
        - -c
        - |
          set -eu

          echo "=== TEMPORARY PVC ==="
          df -h /temp
          du -sh /temp

          echo "=== NEW GRAFANA PVC ==="
          df -h /new
          du -sh /new

          echo "=== COPYING DATA ==="
          cp -a /temp/. /new/

          echo "=== COPY COMPLETE ==="
          echo "Temporary:"
          du -sh /temp

          echo "New Grafana:"
          du -sh /new

          echo "=== FILE COUNTS ==="
          echo -n "TEMP: "
          find /temp -type f | wc -l
          echo -n "NEW:  "
          find /new -type f | wc -l

          echo "Migration complete."

      volumeMounts:
        - name: temp
          mountPath: /temp
          readOnly: true

        - name: new
          mountPath: /new

  volumes:
    - name: temp
      persistentVolumeClaim:
        claimName: grafana-migration

    - name: new
      persistentVolumeClaim:
        claimName: kube-prometheus-stack-grafana
```

Apply:
```bash
kubectl apply -f copyfromtemp.yaml -n monitoring
```

7. Delete Migration Pod and Temporary PVC
```bash
kubectl delete pod grafana-volume-migration-2 -n monitoring
```
Delete the temporary PVC:
```bash
kubectl delete pvc grafana-migration -n monitoring
```
