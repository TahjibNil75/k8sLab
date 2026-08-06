# MySQL on Kubernetes

Deploys a single MySQL 8.0 instance into its own `mysql` namespace, with config in a ConfigMap, passwords in a Secret, and data on a PersistentVolume.

Runs on the KIND cluster from [../kind/README.md](../kind/README.md).

## Files

| File | Kind | Purpose |
|---|---|---|
| `namespace.yaml` | Namespace | Creates the `mysql` namespace |
| `configmap.yaml` | ConfigMap | Non-secret env: `MYSQL_DATABASE`, `MYSQL_USER` |
| `secret.yaml` | Secret | Passwords: `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD` |
| `pv.yaml` | PersistentVolume | 2Gi hostPath volume at `/data/mysql`, class `manual`, pinned to one node |
| `pvc.yaml` | PersistentVolumeClaim | Claims 2Gi from `mysql-pv` |
| `deployment.yaml` | Deployment | 1 replica of `mysql:8.0`, data at `/var/lib/mysql` |
| `service.yaml` | Service (ClusterIP) | Stable DNS name `mysql` on port 3306 |

## Credentials

| Setting | Value | Source |
|---|---|---|
| Database | `appdb` | ConfigMap |
| User | `appuser` | ConfigMap |
| User password | `user123` | Secret |
| Root password | `root123` | Secret |

To change a password, edit the plain text value in `secret.yaml` before you apply it:

```yaml
stringData:
  MYSQL_ROOT_PASSWORD: root123
  MYSQL_PASSWORD: user123
```

`stringData` accepts plain text and Kubernetes base64 encodes it during creation. The older `data:` field expects you to encode the value yourself:

```bash
echo -n "root123" | base64      # cm9vdDEyMw==
```

Both end up as the same stored Secret. Read one back with:

```bash
kubectl get secret mysql-secret -n mysql -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
```

That decoding is the point: a Secret is **encoded, not encrypted**. Anyone who can read it can read the password.

MySQL only reads these passwords **once** — on first startup, when it initializes an empty data directory. Set them before your first `kubectl apply`. Changing the Secret afterwards does nothing, because the data directory already exists; to start over with new passwords, run the [Cleanup](#cleanup) steps and apply again.

## Apply

Order matters — the namespace must exist first, and the Deployment needs the ConfigMap, Secret, and PVC to already be there.

```bash
kubectl apply -f namespace.yaml

kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

If you already applied an earlier version of these manifests, delete the old PVC and PV before re-applying — most of a PV's spec (`storageClassName`, `nodeAffinity`, `hostPath`) is immutable once created, so `kubectl apply` on top of the existing object is rejected:

```bash
kubectl delete -f deployment.yaml -f pvc.yaml
kubectl delete -f pv.yaml
```

## Verify

```bash
kubectl get all -n mysql
kubectl get pv
kubectl get pvc -n mysql
kubectl logs -n mysql -l app=mysql
```

The PVC should show `Bound` and the pod `Running`.

### Storage note

Both `pv.yaml` and `pvc.yaml` set `storageClassName: manual`, so the PVC binds to `mysql-pv` and nothing is provisioned dynamically. Four pieces make that work:

| Field | Where | Why |
|---|---|---|
| `storageClassName: manual` | both | An empty value would let the default `standard` class win, and local-path would provision a different volume — leaving `mysql-pv` `Available` and unused. `manual` is not a real StorageClass object; any name that matches on both sides and isn't a real class disables dynamic provisioning. |
| `claimRef` / `volumeName` | PV / PVC | Reserves the pair for each other, so no other PVC can claim `mysql-pv` first. |
| `type: DirectoryOrCreate` | PV | Creates `/data/mysql` on the node instead of failing the mount when it doesn't exist. |
| `nodeAffinity` | PV | Pins the volume to one node — see below. |

**The node pin matters on a multi-node cluster.** `hostPath` is a directory on whichever node the pod runs on. With two workers and no affinity, a rescheduled pod can land on the other worker, find an empty `/data/mysql`, and initialize a fresh database — the old data is still there, just on the other node. The `nodeAffinity` block pins the PV to `my-kind-cluster-worker`, and the scheduler then places the pod on that node too.

That node name comes from the cluster name in [../kind/README.md](../kind/README.md) (`--name my-kind-cluster`). If your cluster is named differently, check the real names and update `pv.yaml`:

```bash
kubectl get nodes
```

The Deployment uses `strategy: Recreate` for the same reason: the default rolling update would start a second pod before the old one is gone, and two `mysqld` processes on one data directory corrupt it.

## Test Connection

Start a throwaway MySQL client pod in the namespace:

```bash
kubectl run mysql-client --rm -it --image=mysql:8 -n mysql -- bash
```

Inside the pod, connect over the Service DNS name:

```bash
mysql -h mysql -u root -p        # root123
mysql -h mysql -u appuser -p     # user123, appdb
```

Then:

```sql
SHOW DATABASES;
USE appdb;
```

## Connect from your machine

```bash
kubectl port-forward svc/mysql 3306:3306 -n mysql
```

MySQL is then reachable at `127.0.0.1:3306` while that command runs.

## Cleanup

```bash
kubectl delete -f service.yaml -f deployment.yaml -f pvc.yaml -f configmap.yaml -f secret.yaml
kubectl delete -f pv.yaml
kubectl delete -f namespace.yaml
```

The PV uses `persistentVolumeReclaimPolicy: Retain`, so data under `/data/mysql` on `my-kind-cluster-worker` survives deleting the PVC.

Because of `Retain`, re-applying `pv.yaml` and `pvc.yaml` after a delete does **not** rebind — the recreated PV is `Released`/`Available` but the old directory is still on disk with the old passwords baked in. To start genuinely clean, wipe the directory on the node as well:

```bash
docker exec my-kind-cluster-worker rm -rf /data/mysql
```

## Resource Dependency Diagram

```
Namespace (mysql)
│
├── ConfigMap ──── MYSQL_DATABASE, MYSQL_USER
│
├── Secret ─────── MYSQL_ROOT_PASSWORD, MYSQL_PASSWORD
│
├── PVC ──────────── binds ──────────── PV (hostPath /data/mysql)
│
└── Deployment
        │  uses ConfigMap + Secret as env
        │  mounts PVC at /var/lib/mysql
        │
        └── Pod (app: mysql)
                │
                └── Service (ClusterIP, port 3306)
```
