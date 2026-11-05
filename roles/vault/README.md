# hashicorp/vault

## Vault Initialization and Unseal (UI Guide)

This guide explains how to initialize and unseal HashiCorp Vault deployed on k3s using Helm, **via the UI**, up to the point of logging in.

* *Initialization is always required on first deployment.*
* *Manual unseal is required after every restart unless auto-unseal is configured.*
* *For simple in-cluster secrets usage, many teams just initialize once and keep keys safe.*


---

### 1. Access the Vault UI

Open your browser and navigate to your Vault UI:

### 2. Initialize Vault

1. On the main screen, you will see: `Let's set up the initial set of root keys that you will need in case of an emergency.`
2. Enter the following values:

- **Key Shares:** `5`  
- **Key Threshold:** `3`

3. Click **Initialize**.

4. Vault will generate:

- **5 unseal keys** (any 3 are needed to unseal Vault)  
- **Initial root token**

5. **Important:** Store the **unseal keys** and **root token** securely offline or in a safe password manager.

---

### 3. Unseal Vault

Vault is initially **sealed**. To unseal:

1. Enter **Unseal Key Portion 1** in the UI form and click **Unseal**.  
2. Enter **Unseal Key Portion 2** and click **Unseal**.  
3. Enter **Unseal Key Portion 3** and click **Unseal**.  

Once 3 keys are entered, Vault will become **unsealed**.

---

### 4. Login to Vault

1. In the UI, click **Login**.  
2. Select **Token** as the authentication method.  
3. Enter the **Initial Root Token** generated during initialization.  

Vault is now **unsealed and ready for use**.

---
More docs on `ansible-roles/roles/helm_vault/README.md`

---

# Vault PVC Migration: `CSI` → `ROOK`

## Description

This procedure migrates the Vault persistent volume used by `data-vault-0` from the old `ceph-rbd` StorageClass to the new `ceph-block` StorageClass.

This migration is done using a temporary PVC because the current Helm configuration does not support using an existing PVC for Vault in the required way.



1. Stop Vault

Stop Vault and the related vault components:

```bash
kubectl scale --replicas=0 -n vault \
  statefulset.apps/vault
```

2. Create Temporary PVC

Create `temp-pvc.yaml`:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vault-migration
  namespace: vault
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ceph-block
  volumeMode: Filesystem

```

Apply:

```bash
kubectl apply -n vault -f temp-pvc.yaml
```



3. Copy Old PVC → Temporary PVC
Create `migration.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-volume-migration
  namespace: vault
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
        claimName: data-vault-0
    - name: new
      persistentVolumeClaim:
        claimName: vault-migration

```

Apply:

```bash
kubectl -n vault apply -f migration.yaml
```

4. Delete the Old Vault PVC
Once the data has been successfully copied to `vault-migration`, delete the old Vault PVC:

```bash
kubectl delete pvc -n vault data-vault-0

# and delete the migration pod
kubectl delete pod -n vault vault-volume-migration
```

5. Redeploy Vault to create new PVC using new Storageclass

6. Copy Temporary PVC → New Vault PVC
Create `copyfromtemp.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-volume-migration-2
  namespace: vault
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

          echo "=== NEW vault PVC ==="
          df -h /new
          du -sh /new

          echo "=== COPYING DATA ==="
          cp -a /temp/. /new/

          echo "=== COPY COMPLETE ==="
          echo "Temporary:"
          du -sh /temp

          echo "New vault:"
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
        claimName: vault-migration

    - name: new
      persistentVolumeClaim:
        claimName: data-vault-0
```

Apply:
```bash
kubectl apply -f copyfromtemp.yaml -n vault
```

7. Delete Migration Pod and Temporary PVC
```bash
kubectl delete pod vault-volume-migration-2 -n vault
```
Delete the temporary PVC:
```bash
kubectl delete pvc vault-migration -n vault
```
