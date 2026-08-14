# Module 4 — Advanced administration, install, virtualization (EX380 · EX432 · EX316)

Module 4 is where you stop being a user of OpenShift and start being its
**owner**. EX380 is the enterprise administration certification — scaling,
performance, Operators at depth, advanced networking, and compliance. EX432
covers installing clusters from scratch with `openshift-install`. EX316 covers
running VMs alongside containers with OpenShift Virtualization. Two of the
three (EX380, EX316) need a live cluster; EX432 can be studied partly as
knowledge and with a disposable lab install.

## Learning objectives

1. Scale the platform: node pools, machine autoscaling, cluster upgrades with
   `oc adm upgrade`, and performance tuning.
2. Manage Operators at depth: install strategies, subscriptions, and
   troubleshooting operand/CSV states.
3. Configure advanced networking: MetalLB, multiple networks (Multus),
   IngressControllers, and egress/egress-firewall rules.
4. Harden and prove compliance: FIPS, audit logs, SecurityContextConstraints,
   and the Compliance Operator.
5. Install clusters with `openshift-install` (IPI/UPI/agent-based) and
   troubleshoot failed installs.
6. Run, configure, and migrate virtual machines with OpenShift Virtualization
   (`virtctl`, `VMI`, `VirtualMachine`).

## Key concepts

- **Machine API and autoscaling**: `oc get machineset`, `oc scale
  machineset/worker-a --replicas=5`, `oc get machineautoscaler`. The Cluster
  Autoscaler watches pending pods and scales MachineSets within
  MachineAutoscaler bounds.
- **Upgrades**: `oc get clusterversion`, `oc adm upgrade --to-latest` (or
  `--to <version>`), `oc adm upgrade --to-image`. The CVO and MCO sequence
  upgrades; watch `oc get co` during the roll — a `Degraded` operator is the
  first place to look.
- **Machine Config Operator**: owns node OS configuration.
  `oc get machineconfig`, `oc get mc`, and `oc label node --all
  machineconfiguration.openshift.io/role=worker`. Node tuning uses the
  Node Tuning Operator.
- **Operators at depth**: `oc get csv -n openshift-operators`, `oc get
  sub`, `oc get ip` (InstallPlans). `oc describe sub <name> -n
  openshift-operators` and `oc describe csv` diagnose stalled installs —
  usually a missing approval or an image pull failure.
- **MetalLB**: bare-metal/vSphere LoadBalancer — `oc get
  metallb.io` resources; `IPAddressPool` and `L2Advertisement` CRs expose
  Services via `type: LoadBalancer`.
- **Multus**: multiple network interfaces per pod via
  `NetworkAttachmentDefinition` CRs. `oc get network-attachment-definitions`.
- **IngressController and egress**: `oc get ingresscontroller -n
  openshift-ingress-operator`, `oc patch ingresscontroller default ...` to
  set replicas or scope. `EgressFirewall` and `EgressNetworkPolicy` restrict
  outbound traffic per namespace.
- **Compliance**: the Compliance Operator runs OpenSCAP profiles
  (`oc get complianceprofiles`, `oc create -f scan-setting-binding.yaml`),
  producing `ComplianceSuite`/`ComplianceScan` results and remediations.
- **FIPS and hardening**: install with FIPS mode; `oc get nodes --show-labels
  | grep fips`. Audit policy via `oc get audit` custom resources.
- **openshift-install**: `openshift-install create cluster` after writing
  `install-config.yaml`. IPI drives cloud APIs; UPI works on your own
  infrastructure; agent-based targets bare metal/disconnected. Preflight with
  `openshift-install create install-config`, validate with
  `openshift-install wait-for install-complete`.
- **Troubleshooting installs**: `oc get co`, `oc logs -f` on installer pods,
  `openshift-install gather` collects diagnostics. Most install failures are
  DNS, network, or registry mirroring — check `oc get clusteroperators` first.
- **OpenShift Virtualization**: KubeVirt based. CRDs: `VirtualMachine`,
  `VirtualMachineInstance` (VMI), `DataVolume`. `virtctl start/stop/console
  vm`, `virtctl vnc vm`. VMs run as pods on nodes.
- **VM lifecycle**: `virtctl clone`, `virtctl migrate` (live migration),
  `oc create -f vm.yaml`. Storage via `DataVolume`/PVCs; networking via the
  pod network or Multus bridged attachments.

## Hands-on exercises

EX380 and EX316 exercises need CRC (or OKD). EX432 needs a disposable
`openshift-install` run — use a cloud/dev sandbox and read the knowledge bank
alongside.

### Exercise 1 — Autoscaling and upgrade watch

- Task: scale a MachineSet to a fixed size, add a MachineAutoscaler, then
  inspect the ClusterVersion and operator conditions.
- Expected outcome: `oc get machines` shows the scaled count; the
  ClusterAutoscaler and MachineAutoscaler CRs exist; `oc get co` is green.

```sh
oc get machineset -n openshift-machine-api
oc scale machineset worker-a -n openshift-machine-api --replicas=3
oc get machines -n openshift-machine-api
oc get clusterversion
oc adm upgrade
oc get co
```

- Verification (3 worker machines, no degraded operators):

```sh
oc get machines -n openshift-machine-api | grep -c running
oc get co | awk '$3 != "True" { print }'
```

### Exercise 2 — Operator troubleshooting

- Task: install an Operator with a subscription, then read its CSV and
  InstallPlan states.
- Expected outcome: the CSV reaches `Succeeded` and an InstallPlan is
  approved/complete.

```sh
oc get packagemanifests -n openshift-marketplace | grep -i elasticsearch
oc create -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: elasticsearch
  namespace: openshift-operators
spec:
  channel: stable
  name: elasticsearch-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
oc get csv -n openshift-operators
oc get ip -n openshift-operators
```

- Verification (CSV Succeeded, InstallPlan Complete):

```sh
oc get csv -n openshift-operators -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
```

### Exercise 3 — Compliance scan

- Task: create a ScanSettingBinding for the CIS profile and inspect the scan
  result.
- Expected outcome: a ComplianceSuite runs and produces `ComplianceCheckResult`
  resources you can list.

```sh
oc get complianceprofiles -n openshift-compliance
oc create -f - <<'EOF'
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: cis-bind
  namespace: openshift-compliance
profiles:
  - name: rhcos4-e8
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
settingsRef:
  name: default
  kind: ScanSetting
  apiGroup: compliance.openshift.io/v1alpha1
EOF
oc get compliancesuite -n openshift-compliance
oc get compliancecheckresult -n openshift-compliance | head
```

- Verification (suite PHASE DONE, check results listed):

```sh
oc get compliancesuite cis-bind -n openshift-compliance
oc get compliancecheckresult -n openshift-compliance | wc -l
```

### Exercise 4 — EX432 install concepts

- Task: this is mostly conceptual on a lab machine. Write an
  `install-config.yaml`, validate it, and know the install lifecycle.
- Expected outcome: `openshift-install create install-config --dir=lab1`
  produces a config; `openshift-install create cluster --dir=lab1` would
  start an IPI install; the knowledge bank covers the rest.

```sh
mkdir -p lab1
cat > lab1/install-config.yaml <<'EOF'
apiVersion: v1
baseDomain: example.com
controlPlane:
  name: master
  replicas: 3
compute:
- name: worker
  replicas: 3
networking:
  networkType: OVNKubernetes
platform:
  aws:
    region: us-east-1
pullSecret: '{"auths":{}}'
sshKey: |
  ssh-rsa AAAAB3...
EOF
openshift-install create install-config --dir=lab1
openshift-install create cluster --dir=lab1 --log-level=info
```

- Verification (install-config created, installer starts, gather works on
  failure):

```sh
ls lab1/install-config.yaml lab1/auth
openshift-install gather --dir=lab1 2>/dev/null || echo "gather ready"
```

### Exercise 5 — Run a VM with OpenShift Virtualization

- Task: install the OpenShift Virtualization Operator, create a `DataVolume`
  from a disk image, boot a `VirtualMachine`, and connect to its console.
- Expected outcome: the VM reaches `Running`, `virtctl vnc`/`console` opens,
  and live migration is available.

```sh
oc get project openshift-cnv
# install operator + create HyperConverged CR via the web console (OperatorHub)
oc get csv -n openshift-cnv | grep virtualization
virtctl start myvm
oc get vm myvm
oc get vmi myvm
virtctl console myvm
virtctl migrate myvm
```

- Verification (VM Running, migrated successfully):

```sh
oc get vmi myvm -o jsonpath='{.status.phase}{"\n"}'
oc describe vmi myvm | grep -i migration
```

## Test yourself

When you can do Exercises 1–5 without the book, sit simulator attempts:

- **Bank**: `banks/ex380-admin` — **Training** on `scaling-performance` and
  `operators`, then **Mastery** on `security-compliance` and
  `networking-advanced`.
- **Bank**: `banks/ex432` — **Training** on `cluster-installation` and
  `installation-methods`, then **Mastery** on `troubleshooting-install`.
- **Bank**: `banks/ex316` — **Training** on `openshift-virtualization` and
  `vm-lifecycle`, then **Mastery** on `migration-networking`.
- Aim for 80%+ on the Mastery passes before starting Module 5; below that,
  redo Exercises 1–3 (EX380 mechanics) and 5 (VM lifecycle).

## Self-check quiz

1. **Q**: How does the Cluster Autoscaler decide to add a node? **A**: It
   watches for pods stuck in `Pending` that cannot be scheduled on existing
   nodes and scales the annotated MachineSet up to its maximum.
2. **Q**: What does `oc adm upgrade --to-latest` change? **A**: It sets the
   desired version in the ClusterVersion resource, and the CVO moves the
   cluster toward it, upgrading operators in dependency order.
3. **Q**: What is the first thing to check when an Operator install stalls?
   **A**: `oc describe sub` and `oc describe csv` — usually an unapproved
   InstallPlan or an image pull error in the CSV.
4. **Q**: How does a VM differ from a pod on OpenShift Virtualization? **A**: A
   VM is backed by a VirtualMachineInstance (VMI) that runs as a pod on a node
   but boots a guest kernel; it supports live migration via `virtctl migrate`.
5. **Q**: Which object lets a pod use more than one network? **A**: A
   NetworkAttachmentDefinition (Multus), which adds secondary interfaces to
   the pod alongside the default pod network.

## See also

- EX380, EX432, EX316 certification pages on redhat.com (referenced by name).
- docs.openshift.com — Machine API, CVO/MCO, MetalLB, Multus, Compliance
  Operator, OpenShift Virtualization chapters.
- `oc explain machineset`, `man virtctl`, `openshift-install --help`.
- [Module 5 — Automation and data storage (EX430 · EX370)](05-automation-and-storage.md)
  — next: automate the platform and give it durable, protected storage.
