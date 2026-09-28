# [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)


---

```bash
kubectl scale --replicas=0 -n monitoring deployment.apps/kube-prometheus-stack-grafana deployment.apps/kube-prometheus-stack-kube-state-metrics deployment.apps/kube-prometheus-stack-operator
```



Grafana PVC Migration: ceph-rbd → ceph-block
1. Purpose

This runbook describes how to migrate the Grafana persistent volume used by kube-prometheus-stack from the old ceph-rbd StorageClass to the new ceph-block StorageClass.

Current state
Namespace:     monitoring
PVC:           kube-prometheus-stack-grafana
PV:            pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d
Size:          20Gi
StorageClass:  ceph-rbd
AccessMode:    RWO


The new StorageClass is:

ceph-block


ceph-block is currently the default StorageClass.

Migration strategy
OLD PVC
kube-prometheus-stack-grafana
        |
        | ceph-rbd
        |
        | COPY
        v
TEMP PVC
grafana-migration
        |
        | ceph-block
        |
        | COPY
        v
NEW PVC
kube-prometheus-stack-grafana
        |
        | ceph-block
        |
        v
Grafana


The old PV uses Retain, so the underlying volume should remain available after deleting the old PVC.

2. Important safety considerations

Before starting:

Schedule a maintenance window for Grafana.

Grafana must be stopped while copying its data.

Do not manually delete the old PV during the migration.

Do not manually delete the old Ceph RBD image.

Keep the old PV until the migration has been fully verified.

Do not delete the temporary PVC until Grafana has been confirmed working.

The current PV has:

persistentVolumeReclaimPolicy: Retain

3. Verify the current configuration

Check the current PVC:

kubectl get pvc kube-prometheus-stack-grafana \
  -n monitoring \
  -o wide


Check the old PV:

kubectl get pv pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d


Verify the reclaim policy:

kubectl get pv pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d \
  -o jsonpath='{.spec.persistentVolumeReclaimPolicy}{"\n"}'


Expected:

Retain


Check StorageClasses:

kubectl get sc


Expected to include:

ceph-block
ceph-rbd

4. Back up the current PV and PVC definitions

Save the current PV:

kubectl get pv pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d \
  -o yaml > old-grafana-pv.yaml


Save the current PVC:

kubectl get pvc kube-prometheus-stack-grafana \
  -n monitoring \
  -o yaml > old-grafana-pvc.yaml


Keep these files until the migration is complete.

5. Check Helm configuration

The Grafana PVC is managed by the Helm release:

kube-prometheus-stack


Export the current values:

helm get values kube-prometheus-stack \
  -n monitoring \
  -a > kube-prometheus-stack-values.yaml


Inspect the Grafana persistence configuration:

grep -A20 -B5 "persistence:" kube-prometheus-stack-values.yaml


Look for:

grafana:
  persistence:
    enabled: true
    storageClassName: ceph-rbd


If ceph-rbd is explicitly configured, it must eventually be changed to:

grafana:
  persistence:
    enabled: true
    storageClassName: ceph-block


Do not perform the Helm upgrade yet.

6. Stop Grafana

Scale Grafana down:

kubectl scale deployment kube-prometheus-stack-grafana \
  -n monitoring \
  --replicas=0


Verify that the Grafana pod has stopped:

kubectl get pods \
  -n monitoring \
  -l app.kubernetes.io/name=grafana


There should be no running Grafana pod.

7. Create the temporary PVC

Create a file:

vi grafana-migration-pvc.yaml


Contents:

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


Apply:

kubectl apply -f grafana-migration-pvc.yaml


Verify:

kubectl get pvc -n monitoring


Expected:

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
grafana-migration               Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   20Gi       RWO            ceph-block
kube-prometheus-stack-grafana   Bound    pvc-fa177103-...                            20Gi       RWO            ceph-rbd


The temporary PVC must be Bound before continuing.

8. Create the first migration pod

This pod mounts both:

/old = existing ceph-rbd PVC
/new = temporary ceph-block PVC


Create:

vi grafana-volume-migration.yaml


Contents:

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

          echo "=== OLD VOLUME ==="
          df -h /old
          du -sh /old
          echo

          echo "=== NEW VOLUME ==="
          df -h /new
          du -sh /new
          echo

          echo "=== COPYING DATA ==="
          cp -a /old/. /new/

          echo
          echo "=== COPY COMPLETE ==="
          echo "Old volume:"
          du -sh /old
          echo
          echo "New volume:"
          du -sh /new

          echo
          echo "File counts:"
          echo -n "OLD: "
          find /old -type f | wc -l
          echo -n "NEW: "
          find /new -type f | wc -l

          echo
          echo "Migration complete."

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


Apply:

kubectl apply -f grafana-volume-migration.yaml

9. Monitor the first migration

Check the pod:

kubectl get pod grafana-volume-migration \
  -n monitoring


Watch the logs:

kubectl logs -f grafana-volume-migration \
  -n monitoring


The pod should eventually report:

Migration complete.


Verify the pod completed:

kubectl get pod grafana-volume-migration \
  -n monitoring


Expected status:

Completed

10. Verify the temporary copy

Check the size:

kubectl exec -n monitoring grafana-volume-migration -- \
  sh -c 'du -sh /old /new'


Check file counts:

kubectl exec -n monitoring grafana-volume-migration -- \
  sh -c 'find /old -type f | wc -l; find /new -type f | wc -l'


Inspect the copied data:

kubectl exec -n monitoring grafana-volume-migration -- \
  sh -c 'find /new -maxdepth 2 -type f | head -50'


The copied volume should contain the expected Grafana data, such as:

grafana.db
plugins/
png/


depending on the Grafana configuration.

Do not continue until the temporary copy has been verified.

11. Delete the old Grafana PVC

At this point the data exists on:

OLD ceph-rbd volume
TEMP ceph-block volume


Delete only the old PVC:

kubectl delete pvc kube-prometheus-stack-grafana \
  -n monitoring


Verify:

kubectl get pvc -n monitoring


The old Grafana PVC should be gone.

Now check the old PV:

kubectl get pv pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d


Because the reclaim policy is Retain, it should remain, typically in:

Released


state.

Do not delete the old PV.

12. Configure Helm to use ceph-block

Update the Helm values so Grafana uses:

grafana:
  persistence:
    enabled: true
    storageClassName: ceph-block


If your current values contain:

storageClassName: ceph-rbd


change it to:

storageClassName: ceph-block


Then perform your normal Helm upgrade.

Example:

helm upgrade kube-prometheus-stack <chart> \
  -n monitoring \
  -f <your-values-file>


Use the actual chart and values file used by your deployment.

13. Verify the new Grafana PVC

Check:

kubectl get pvc -n monitoring


Expected:

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
grafana-migration               Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   20Gi       RWO            ceph-block
kube-prometheus-stack-grafana   Bound    pvc-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy   20Gi       RWO            ceph-block


Verify the StorageClass explicitly:

kubectl get pvc kube-prometheus-stack-grafana \
  -n monitoring \
  -o jsonpath='{.spec.storageClassName}{"\n"}'


Expected:

ceph-block


Verify the new PV:

kubectl get pv \
  $(kubectl get pvc kube-prometheus-stack-grafana \
    -n monitoring \
    -o jsonpath='{.spec.volumeName}')

14. Copy TEMP → NEW Grafana PVC

Delete the first migration pod:

kubectl delete pod grafana-volume-migration \
  -n monitoring


Create a second migration pod:

vi grafana-volume-migration-2.yaml


Contents:

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

          echo "=== TEMPORARY VOLUME ==="
          df -h /temp
          du -sh /temp
          echo

          echo "=== NEW GRAFANA VOLUME ==="
          df -h /new
          du -sh /new
          echo

          echo "=== COPYING DATA ==="
          cp -a /temp/. /new/

          echo
          echo "=== COPY COMPLETE ==="
          echo "Temporary:"
          du -sh /temp
          echo
          echo "New Grafana:"
          du -sh /new

          echo
          echo "File counts:"
          echo -n "TEMP: "
          find /temp -type f | wc -l
          echo -n "NEW:  "
          find /new -type f | wc -l

          echo
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


Apply:

kubectl apply -f grafana-volume-migration-2.yaml


Watch the logs:

kubectl logs -f grafana-volume-migration-2 \
  -n monitoring


Wait for:

Migration complete.

15. Verify the new Grafana volume

Check volume sizes:

kubectl exec -n monitoring grafana-volume-migration-2 -- \
  sh -c 'du -sh /temp /new'


Check file counts:

kubectl exec -n monitoring grafana-volume-migration-2 -- \
  sh -c 'find /temp -type f | wc -l; find /new -type f | wc -l'


Inspect the new volume:

kubectl exec -n monitoring grafana-volume-migration-2 -- \
  sh -c 'find /new -maxdepth 2 -type f | head -50'

16. Start Grafana

Once the new PVC has been verified, scale Grafana back up:

kubectl scale deployment kube-prometheus-stack-grafana \
  -n monitoring \
  --replicas=1


Watch the pod:

kubectl get pods \
  -n monitoring \
  -l app.kubernetes.io/name=grafana \
  -w


Check Grafana logs:

kubectl logs \
  -n monitoring \
  -l app.kubernetes.io/name=grafana \
  --tail=100


Check for database, filesystem, or permission errors.

17. Verify Grafana

Verify the following:

Grafana starts successfully.

Existing dashboards are present.

Existing users are present.

Existing data sources are present.

Existing folders/configuration are present.

Grafana plugins are available if stored on the PVC.

There are no database corruption errors.

Grafana is using the new PVC.

Check the final PVC:

kubectl get pvc kube-prometheus-stack-grafana \
  -n monitoring \
  -o wide


Expected:

STORAGECLASS
ceph-block

18. Keep the temporary PVC as a safety copy

Do not immediately delete grafana-migration.

Keep it temporarily while verifying Grafana.

At this stage:

OLD ceph-rbd
    |
    | original data
    |
TEMP ceph-block
    |
    | migrated data
    |
NEW ceph-block
    |
    | active Grafana
    v
Grafana


Once Grafana has been successfully verified, delete the temporary PVC:

kubectl delete pvc grafana-migration \
  -n monitoring

19. Old PV cleanup

The old PV should still exist:

kubectl get pv pvc-fa177103-2d49-477f-b0b5-12cf4d25ef4d


Its reclaim policy is:

Retain


Do not remove it until you are completely satisfied with the migration.

Once the migration is confirmed successful, the old PV and underlying Ceph RBD volume can be cleaned up according to your Ceph/CSI operational procedures.

Do not manually delete the underlying Ceph image unless you have confirmed exactly how the CSI volume is managed.

20. Rollback

If Grafana does not work after the migration:

Scale Grafana down.

Do not delete the old PV.

Keep the old retained Ceph volume.

Keep the temporary PVC.

Keep the saved PV/PVC YAML files.

Investigate the new volume.

Restore/recreate the original PVC/PV relationship if required.

The old data should remain available because the old PV uses:

Retain

21. Final expected state

The final PVC should be:

kubectl get pvc -n monitoring


Expected:

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
kube-prometheus-stack-grafana   Bound    pvc-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy   20Gi       RWO            ceph-block


The important change is:

Before:

kube-prometheus-stack-grafana
        |
        v
ceph-rbd
        |
        v
old PV


After:

kube-prometheus-stack-grafana
        |
        v
ceph-block
        |
        v
new PV


The PVC name remains:

kube-prometheus-stack-grafana


so Grafana and the Helm release continue using the same PVC name.

22. Migration checklist

 Verify old PVC is ceph-rbd

 Verify old PV has Retain

 Back up old PV/PVC definitions

 Check Helm configuration

 Stop Grafana

 Create temporary ceph-block PVC

 Verify temporary PVC is Bound

 Copy old PVC → temporary PVC

 Verify temporary data

 Delete old Grafana PVC

 Verify old PV remains Released

 Configure Helm to use ceph-block

 Run Helm upgrade

 Verify new Grafana PVC is ceph-block

 Copy temporary PVC → new Grafana PVC

 Verify new data

 Start Grafana

 Verify dashboards/users/data sources

 Keep temporary PVC until migration is confirmed

 Delete temporary PVC

 Keep old PV until final cleanup is approved

 Clean up old Ceph volume according to operational procedures