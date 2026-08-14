# Admin Module 5 — Storage: PV/PVC/StorageClasses; Operators and their lifecycle

The final admin module covers the two remaining exam pillars: persistent
storage (how applications keep state across restarts) and Operators (how the
platform itself is extended and managed). Both are exercised heavily in
EX280-style tasks and in the developer track's workloads.

## Learning objectives

1. Describe PV, PVC, and StorageClass and how they bind and provision.
2. Consume storage in a workload and verify persistence across pod restarts.
3. Configure access modes, reclaim policies, and volume binding modes.
4. Install, watch, approve, and remove an Operator through OLM, and identify
   the Custom Resources it manages.

## Key concepts

- **PV / PVC / StorageClass**: a **PersistentVolume** is cluster-scoped
  capacity; a **PersistentVolumeClaim** is a namespaced request for capacity; a
  **StorageClass** describes *how* capacity is provisioned and *what* policies
  apply. The scheduler/controller matches a PVC to a PV (or asks the
  provisioner for one).
- **Access modes**: `ReadWriteOnce` (single node), `ReadWriteMany` (many
  nodes), `ReadWriteOncePod` (a single pod). A PV binds only to PVCs whose
  access mode it satisfies.
- **Volume modes**: `Filesystem` (a mounted filesystem) vs `Block` (raw block
  device) — selected per PVC, not per storage class.
- **Reclaim policy**: what happens to a PV when its PVC is deleted: `Retain`
  (leave data, PV goes `Released`), `Delete` (provisioner deletes the backing
  volume), `Recycle` (deprecated). Set on the StorageClass or PV.
- **StorageClass fields**: `provisioner`, `reclaimPolicy`,
  `volumeBindingMode`, `allowVolumeExpansion`, plus parameters. The default
  class carries the annotation
  `storageclass.kubernetes.io/is-default-class: "true"` and is used whenever a
  PVC omits `storageClassName`.
- **Binding modes**: `Immediate` (bind at claim time) vs
  `WaitForFirstConsumer` (bind when a pod consumes it — enables topology-aware
  scheduling and lazy provisioning).
- **Dynamic provisioning**: most classes provision on demand. On OpenShift
  Local the hostpath CSI provisioner is the default; name it explicitly in
  commands and verify with `oc get storageclass`.
- **Consumption**: a workload mounts a claim via `volumeMounts` +
  `volumes.persistentVolumeClaim`. Data written to the mount survives pod
  deletion because it lives in the volume, not the container.
- **Operators and OLM**: the **Operator Lifecycle Manager** installs and runs
  Operators. Key resources: `PackageManifest` (catalog entry), `Subscription`
  (desired operator + channel + approval strategy), `InstallPlan` (what gets
  installed, pending approval), `ClusterServiceVersion` (CSV — the installed
  operator version and its CRD/API surface), `OperatorGroup` (which namespaces
  the operator watches).
- **Approval strategy**: `Automatic` installs upgrades immediately;
  `Manual` queues an InstallPlan for you to approve with
  `oc patch installplan <ip> --type merge --patch '{"spec":{"approved":true}}'`.
- **Catalog surface**: `oc get packagemanifest` lists available operators
  (the embedded catalog on OpenShift Local is smaller than the full
  OperatorHub). `oc describe packagemanifest <name>` shows channels and the
  default install namespace.
- **Operand vs operator**: the Operator is the control loop; the **operand**
  is the application/CR it manages. Installing an operator usually lets you
  create new CustomResources — verify with `oc api-resources` afterward.
- **Uninstall**: delete the Subscription, wait for the CSV/InstallPlan to
  clear, then delete the OperatorGroup; confirm with `oc get csv -A`.

## Hands-on exercises

### Exercise 1 — Inspect the default StorageClass and create a PVC

- Task: identify the default StorageClass, then create a 1 GiB `ReadWriteOnce`
  PVC named `data-vol` in `alpha` and watch it bind.
- Expected outcome: `oc get pvc data-vol` shows `Bound` to a real PV within a
  few seconds.

```sh
oc project alpha
oc get storageclass
oc get sc -o jsonpath='{range .items[*]}{.metadata.name}{" default="}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
oc create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-vol
  namespace: alpha
spec:
  accessModes: [ReadWriteOnce]
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
EOF
```

- Verification:

```sh
oc get pvc data-vol -o wide
oc get pv -o wide
oc get pvc data-vol -o jsonpath='{.spec.volumeName}{"\n"}'
```

### Exercise 2 — Consume the claim and prove persistence

- Task: deploy `db-store` mounting `data-vol` at `/data`, write a marker file,
  delete the pod, and confirm the marker survives.
- Expected outcome: after deleting the pod (the Deployment recreates it) the
  file `/data/marker.txt` still exists.

```sh
oc create deployment db-store --image=registry.access.redhat.com/ubi9/ubi-minimal:latest -- /bin/sh -c 'sleep 3600'
oc set volume deployment/db-store --add --name=data --type=pvc --claim-name=data-vol --mount-path=/data
oc exec deployment/db-store -- touch /data/marker.txt
```

- Verification:

```sh
oc exec deployment/db-store -- ls -la /data/marker.txt
oc delete pod -l app=db-store
oc wait --for=condition=Available deployment/db-store --timeout=120s
oc exec deployment/db-store -- cat /data/marker.txt
```

### Exercise 3 — WaitForFirstConsumer binding

- Task: create StorageClass `lazy-sc` with `volumeBindingMode:
  WaitForFirstConsumer` (reusing the local provisioner), claim from it, and
  observe that the claim stays `Pending` until a pod uses it.
- Expected outcome: `oc get pvc lazy-claim` reports `Pending` (no bound PV),
  and after scheduling a pod the claim binds.

```sh
oc get sc -o jsonpath='{.items[0].provisioner}{"\n"}'
oc apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lazy-sc
provisioner: crc-csi-hostpath-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
oc create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lazy-claim
  namespace: alpha
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: lazy-sc
  resources:
    requests:
      storage: 100Mi
EOF
```

- Verification:

```sh
oc get pvc lazy-claim -o wide
oc describe pvc lazy-claim | grep -i "waiting for first consumer"
```

### Exercise 4 — Manual PV with Retain policy

- Task: create a hostPath PV `manual-pv` with `reclaimPolicy: Retain` in the
  `beta` project, bind a claim to it, and verify the PV stays `Released`
  (with data) after the claim is deleted.
- Expected outcome: PV `manual-pv` exists with phase `Available`, then `Bound`
  to your claim, then `Released` after deletion — still keeping the backing
  path.

```sh
oc project beta
oc apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 100Mi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/manual-pv-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [crc-x8mxw-master-0]
EOF
oc create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-claim
  namespace: beta
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Mi
EOF
```

- Verification:

```sh
oc get pv manual-pv -o wide
oc delete pvc manual-claim -n beta
oc get pv manual-pv -o jsonpath='{.status.phase}{"\n"}'
```

> The node name `crc-x8mxw-master-0` is OpenShift Local specific — on a
> different cluster substitute the output of `oc get nodes -o name`; a
> single-node hostPath PV only works because control plane and worker share one
> host.

### Exercise 5 — Install an Operator through OLM

- Task: find an operator in the catalog, create a Subscription, and walk it
  through InstallPlan → CSV → available operand API.
- Expected outcome: `oc get csv` shows `Succeeded` for your operator, and a new
  custom resource kind appears in `oc api-resources`.

```sh
oc get packagemanifest
oc get packagemanifest | grep -i metrics
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metrics-server
  namespace: openshift-operators
spec:
  channel: stable
  name: metrics-server
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
EOF
oc get installplan -n openshift-operators -w
```

- Verification:

```sh
oc get csv -n openshift-operators -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
oc api-resources | grep -i metrics
oc patch installplan $(oc get installplan -n openshift-operators -o jsonpath='{.items[0].metadata.name}') -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'
oc wait --for=jsonpath='{.status.phase}'=Succeeded csv/metrics-server-0.7.0 -n openshift-operators --timeout=180s
```

> The exact package name, channel, and CSV version depend on the embedded
> catalog of your release. If `metrics-server` is absent on your cluster, pick
> any other `oc get packagemanifest` entry and adapt the names — the OLM flow
> (Subscription → InstallPlan → CSV) is identical.

## Test yourself

When Exercises 1–5 are smooth:

- **Domains**: `storage` and `operators` in the `ex280-admin` bank. These are
  separate domains, so sit **two focused attempts** — one per domain — or one
  full attempt if your bank draws them together.
- **First pass**: **Training** mode on each domain — read every explanation.
- **Second pass**: **Mastery** mode, timed, on each domain.
- **Spiral**: re-sit a full-bank **Training** attempt now that every domain is
  introduced, then a full-bank **Mastery** attempt. You have covered the whole
  EX280 surface; weeks 6–8 of the [schedule](../README.md) are drill and dress
  rehearsal.

Weakness on `storage`? Redo Exercises 1–2 (binding + consumption). Weakness on
`operators`? Redo Exercise 5 and practice the `oc patch installplan` approval
until it is muscle memory — it is a classic live-task.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Admin path index](README.md) — this completes the admin path. Next
  suggested step: the full-bank Mastery and Exam-mode rehearsal in the global
  schedule.
