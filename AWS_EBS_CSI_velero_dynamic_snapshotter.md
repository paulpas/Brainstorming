Below is a **complete, self‑contained solution** that lives **inside the EKS cluster**:

1. **RBAC** – a ServiceAccount that can create `Backup` objects.  
2. **ConfigMap** – holds a tiny Python script that talks to the Kubernetes API (no `kubectl` binary needed).  
3. **Job** – runs the script on demand (you can also turn it into a CronJob, an Argo‑Workflow step, a Tekton task, etc.).  

Everything is < 30 lines of YAML + a ~30‑line Python script.  
You can trigger a snapshot by simply creating a `Job` (or by calling the script from any other pod).

---

## 1️⃣ RBAC – give a pod permission to create Velero backups

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: velero-trigger-sa
  namespace: velero          # <-- same namespace where Velero is installed
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: velero-trigger-role
  namespace: velero
rules:
- apiGroups: ["velero.io"]
  resources: ["backups"]
  verbs: ["create","get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: velero-trigger-rb
  namespace: velero
subjects:
- kind: ServiceAccount
  name: velero-trigger-sa
  namespace: velero
roleRef:
  kind: Role
  name: velero-trigger-role
  apiGroup: rbac.authorization.k8s.io
```

*Why a **Role** (not a ClusterRole)?*  
`Backup` is a namespaced CRD, so a namespace‑scoped role is sufficient.

---

## 2️⃣ ConfigMap – the “trigger” script

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-trigger-script
  namespace: velero
data:
  trigger_snapshot.py: |
    #!/usr/bin/env python3
    """
    Create a Velero Backup that snapshots all PVCs in a given namespace.
    No Velero CLI is used – we talk directly to the Kubernetes API.
    """
    import os, sys, datetime, uuid, json
    from kubernetes import client, config

    # ---------- CONFIG (override with env vars) ----------
    TARGET_NS   = os.getenv("TARGET_NS", "default")          # PVC namespace to snapshot
    BACKUP_NS   = os.getenv("VELERO_NS", "velero")          # where Velero CRDs live
    PREFIX      = os.getenv("BK_PREFIX", "eks-snap")
    # -----------------------------------------------------

    # Load in‑cluster config (works when the pod runs inside the cluster)
    config.load_incluster_config()
    api = client.CustomObjectsApi()

    # Build a unique backup name
    now = datetime.datetime.utcnow().strftime("%Y-%m-%dT%H-%M-%SZ")
    uid = str(uuid.uuid4())[:8]
    backup_name = f"{PREFIX}-{now}-{uid}"

    # Backup manifest – identical to what `velero backup create …` would generate
    backup_body = {
        "apiVersion": "velero.io/v1",
        "kind": "Backup",
        "metadata": {"name": backup_name, "namespace": BACKUP_NS},
        "spec": {
            "includedNamespaces": [TARGET_NS],
            "includedResources": ["persistentvolumeclaims"],
            "snapshotVolumes": True,
            "ttl": "720h"
        }
    }

    # Create the Backup CR
    try:
        api.create_namespaced_custom_object(
            group="velero.io",
            version="v1",
            namespace=BACKUP_NS,
            plural="backups",
            body=backup_body,
        )
        print(f"✅ Backup {backup_name} created – snapshots will start now")
    except client.exceptions.ApiException as e:
        print(f"❌ Failed to create backup: {e}", file=sys.stderr)
        sys.exit(1)
```

*Dependencies* – the container image will ship the `kubernetes` Python client (≈ 2 MB).  

---

## 3️⃣ Job that runs the script (one‑off trigger)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: velero-trigger-snapshot
  namespace: velero
spec:
  backoffLimit: 2
  template:
    spec:
      serviceAccountName: velero-trigger-sa
      restartPolicy: Never
      containers:
      - name: trigger
        image: python:3.11-slim          # tiny base image
        command: ["python", "/scripts/trigger_snapshot.py"]
        env:
        - name: TARGET_NS
          value: "default"               # <-- change to the PVC‑namespace you want
        - name: VELERO_NS
          value: "velero"
        - name: BK_PREFIX
          value: "eks-snap"
        volumeMounts:
        - name: script
          mountPath: /scripts
      volumes:
      - name: script
        configMap:
          name: velero-trigger-script
          items:
          - key: trigger_snapshot.py
            path: trigger_snapshot.py
```

**How to fire a snapshot**

```bash
# From any machine that can talk to the cluster (kubectl, Argo, CI, etc.)
kubectl create job --from=cronjob/velero-trigger-snapshot dummy-$(date +%s) -n velero
# or simply:
kubectl apply -f job-trigger.yaml   # (the YAML above)
```

When the Job starts, the Python script creates a `Backup` CR. Velero’s controller sees the new object and immediately begins the CSI‑snapshot workflow for **every PVC in `TARGET_NS`**. The process is identical to running:

```bash
velero backup create <name> --include-namespaces <ns> --include-resources pvc --snapshot-volumes
```

but **without ever invoking the Velero CLI**.

---

## 4️⃣ Optional: Turn it into a CronJob (daily, hourly, etc.)

If you prefer a scheduled snapshot instead of a manual Job, just replace the `Job` with a `CronJob`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: velero-scheduled-snapshot
  namespace: velero
spec:
  schedule: "0 0 * * *"          # every night at 00:00 UTC
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: velero-trigger-sa
          restartPolicy: Never
          containers:
          - name: trigger
            image: python:3.11-slim
            command: ["python", "/scripts/trigger_snapshot.py"]
            env:
            - name: TARGET_NS
              value: "default"
            - name: VELERO_NS
              value: "velero"
            - name: BK_PREFIX
              value: "eks-snap"
            volumeMounts:
            - name: script
              mountPath: /scripts
          volumes:
          - name: script
            configMap:
              name: velero-trigger-script
              items:
              - key: trigger_snapshot.py
                path: trigger_snapshot.py
```

---

## 5️⃣ How this avoids the “local CLI” problem

| What you normally do | What we do instead |
|----------------------|--------------------|
| Run `velero backup create …` on a laptop or a CI runner (requires the Velero binary, kubeconfig, etc.) | **Create a `Backup` CR** via the Kubernetes API from *inside* the cluster. No external binary, no kubeconfig file, no network‑hop. |
| Need to manage IAM credentials for the CLI | The pod runs with a **ServiceAccount** that already has the required RBAC (`velero.io` `backups` create). |
| Hard to batch or schedule without extra tooling | Use native **Job / CronJob** objects, or embed the call in any controller you already run (Argo, Tekton, custom operator). |

---

## 6️⃣ Full “copy‑paste” bundle (everything together)

```yaml
# 1️⃣ ServiceAccount + RBAC
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: velero-trigger-sa
  namespace: velero
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: velero-trigger-role
  namespace: velero
rules:
- apiGroups: ["velero.io"]
  resources: ["backups"]
  verbs: ["create","get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: velero-trigger-rb
  namespace: velero
subjects:
- kind: ServiceAccount
  name: velero-trigger-sa
  namespace: velero
roleRef:
  kind: Role
  name: velero-trigger-role
  apiGroup: rbac.authorization.k8s.io

# 2️⃣ ConfigMap with the Python trigger script
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-trigger-script
  namespace: velero
data:
  trigger_snapshot.py: |
    #!/usr/bin/env python3
    import os, sys, datetime, uuid
    from kubernetes import client, config
    TARGET_NS   = os.getenv("TARGET_NS", "default")
    BACKUP_NS   = os.getenv("VELERO_NS", "velero")
    PREFIX      = os.getenv("BK_PREFIX", "eks-snap")
    config.load_incluster_config()
    api = client.CustomObjectsApi()
    now = datetime.datetime.utcnow().strftime("%Y-%m-%dT%H-%M-%SZ")
    uid = str(uuid.uuid4())[:8]
    backup_name = f"{PREFIX}-{now}-{uid}"
    backup_body = {
        "apiVersion": "velero.io/v1",
        "kind": "Backup",
        "metadata": {"name": backup_name, "namespace": BACKUP_NS},
        "spec": {
            "includedNamespaces": [TARGET_NS],
            "includedResources": ["persistentvolumeclaims"],
            "snapshotVolumes": True,
            "ttl": "720h"
        }
    }
    try:
        api.create_namespaced_custom_object(
            group="velero.io",
            version="v1",
            namespace=BACKUP_NS,
            plural="backups",
            body=backup_body,
        )
        print(f"✅ Backup {backup_name} created")
    except client.exceptions.ApiException as e:
        print(f"❌ Failed: {e}", file=sys.stderr)
        sys.exit(1)

# 3️⃣ Example one‑off Job (run `kubectl apply -f this.yaml` to fire a snapshot)
---
apiVersion: batch/v1
kind: Job
metadata:
  name: velero-trigger-snapshot
  namespace: velero
spec:
  backoffLimit: 2
  template:
    spec:
      serviceAccountName: velero-trigger-sa
      restartPolicy: Never
      containers:
      - name: trigger
        image: python:3.11-slim
        command: ["python", "/scripts/trigger_snapshot.py"]
        env:
        - name: TARGET_NS
          value: "default"          # <-- change as needed
        - name: VELERO_NS
          value: "velero"
        - name: BK_PREFIX
          value: "eks-snap"
        volumeMounts:
        - name: script
          mountPath: /scripts
      volumes:
      - name: script
        configMap:
          name: velero-trigger-script
          items:
          - key: trigger_snapshot.py
            path: trigger_snapshot.py
```

Apply the whole file once:

```bash
kubectl apply -f velero-trigger-bundle.yaml
```

Then **fire a snapshot** whenever you like:

```bash
kubectl create job --from=cronjob/velero-trigger-snapshot manual-$(date +%s) -n velero
# or simply:
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: manual-snap-$(date +%s)
  namespace: velero
spec:
  template:
    spec:
      serviceAccountName: velero-trigger-sa
      restartPolicy: Never
      containers:
      - name: trigger
        image: python:3.11-slim
        command: ["python", "/scripts/trigger_snapshot.py"]
        env:
        - name: TARGET_NS
          value: "default"
        volumeMounts:
        - name: script
          mountPath: /scripts
      volumes:
      - name: script
        configMap:
          name: velero-trigger-script
EOF
```

Velero will immediately start CSI snapshots for **all PVCs in the chosen namespace**, respecting the same rate‑limit handling that the CLI does (it uses the same controller code under the hood).

---

## 7️⃣ What if you need *staggering* (rate‑limit safety)?

The **Velero controller** already respects the AWS EBS `CreateSnapshot` rate limit (it backs off and retries).  
If you still want to be extra‑conservative, you can add a tiny “sleep” loop inside the Python script before creating each backup, e.g.:

```python
import time
MAX_PER_MIN = int(os.getenv("MAX_PER_MIN", "80"))
INTERVAL = 60.0 / MAX_PER_MIN
# after each successful create:
time.sleep(INTERVAL)
```

Because the script creates **one Backup object per PVC**, the interval guarantees you never fire more than `MAX_PER_MIN` snapshot requests in any minute.

---

## 8️⃣ TL;DR checklist

| ✅ Done? | Item |
|---------|------|
| ✅ | ServiceAccount + Role/RoleBinding that can `create` Velero `Backup` CRs. |
| ✅ | ConfigMap with a tiny Python script that talks to the Kubernetes API (no `kubectl`/Velero CLI). |
| ✅ | Job (or CronJob) that mounts the script and runs it. |
| ✅ | One‑line `kubectl create job …` (or any controller) to fire a snapshot on demand. |
| ✅ | Optional throttling control via `MAX_PER_MIN` env var. |
| ✅ | All resources live in the same cluster – no external binaries, no extra IAM keys. |

You now have a **pure‑Kubernetes, API‑driven way** to trigger Velero CSI snapshots on EKS without ever invoking the local Velero CLI. 🎉
