# Admin Module 1 — Cluster architecture, install concepts, and the oc CLI

Module 1 grounds you in how OpenShift 4 is put together and how you talk to it.
Everything later in the path — RBAC, SCCs, networking, storage, Operators —
happens through the same `oc`-driven workflow you learn here.

## Learning objectives

1. Explain the OpenShift 4 control plane: API server, etcd, scheduler,
   controllers, and how cluster operators own the platform.
2. Distinguish the OpenShift install models (IPI/UPI, assisted, agent-based)
   and describe how OpenShift Local and OKD relate to the commercial product.
3. Authenticate with `oc`, switch contexts, and inspect your own identity.
4. Query the API with `oc get`/`oc describe`/`oc explain`, including JSONPath
   and label selectors, and check cluster health via cluster operators.

## Key concepts

- **Control plane**: the API server (`kube-apiserver`) is the front door;
  etcd stores all state; the scheduler places pods; controllers reconcile
  desired state. In OpenShift 4 these run as static pods on control-plane
  nodes, not as systemd services.
- **Cluster Operators**: OpenShift 4 is operator-driven. Components such as
  DNS, Ingress, monitoring, and the API server are each owned by a cluster
  operator (`oc get clusteroperators`). An operator reports `Available`,
  `Progressing`, and `Degraded` conditions.
- **Nodes**: workers run user workloads; control-plane nodes run the platform.
  OpenShift Local (CRC) collapses both onto one node, which is why it needs
  4+ vCPU and 16+ GiB RAM.
- **Install models**: **IPI** (Installer Provisioned Infrastructure) creates
  its own cloud VMs and the Machine API manages them; **UPI** (User Provided
  Infrastructure) lets you install on infrastructure you manage yourself;
  the **agent-based** installer handles disconnected/bare-metal scenarios.
  `openshift-install create cluster` drives an install.
- **oc vs kubectl**: `oc` is a superset of `kubectl`. Prefer `oc` everywhere
  — it also speaks OpenShift-native APIs (routes, buildconfigs, projects,
  scc).
- **Auth and context**: `oc login` exchanges credentials for a token stored in
  your kubeconfig. `oc whoami` shows your user, `oc whoami -t` prints the raw
  token. `oc config get-contexts` lists contexts; `oc config use-context`
  switches; `oc project` shows/sets the current project.
- **Projects vs namespaces**: a project is a namespace plus extra metadata
  (description, display name, owner annotations). `oc new-project` creates
  both. `oc project <name>` switches your active project.
- **Querying**: `oc get` lists resources, `oc describe` shows detail and
  events, `oc explain <resource>` documents a resource's fields, `oc
  api-resources` lists the APIs the cluster serves.
- **Output control**: `-o wide`, `-o yaml`, `-o json`, and `-o
  jsonpath='{...}'` shape results for scripting; `-l` filters by label.
- **Cluster health**: `oc get clusterversion` shows version and upgrade state;
  `oc get co` shows every cluster operator; a `Degraded=True` operator
  explains why via `oc describe co <name>`.
- **Mutation model**: `oc create` makes new objects, `oc apply` reconciles
  from a manifest file (declarative), `oc edit` opens the live object. OpenShift
  admission controllers mutate and validate on the way in — you will see this
  matter in the SCC module.

## Hands-on exercises

All exercises assume you are logged in as `kubeadmin` on your local cluster.

### Exercise 1 — Log in and verify your identity

- Task: authenticate against your cluster's API and confirm who you are and
  what version you are running.
- Expected outcome: `oc whoami` prints `kubeadmin`, `oc version` prints client
  and server versions, and the current project is visible.

```sh
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
oc whoami
oc whoami -t
oc version
oc project
```

- Verification:

```sh
oc whoami
oc auth can-i --list -n default | head
```

### Exercise 2 — Explore the API surface

- Task: list the resource kinds the cluster understands, then inspect the
  documentation for the `route` resource.
- Expected outcome: `oc api-resources` includes `routes.route.openshift.io`
  (or similar), and `oc explain route` prints the field reference without
  error.

```sh
oc api-resources | grep -i route
oc explain route
oc explain route.spec.host
```

- Verification:

```sh
oc api-resources --namespaced=false | wc -l
```

### Exercise 3 — Inspect cluster health

- Task: report cluster version, node status, and the condition of every
  cluster operator.
- Expected outcome: nodes are `Ready`; cluster operators are all
  `Available=True` (on a healthy fresh cluster); the cluster version shows the
  running release.

```sh
oc get clusterversion
oc get nodes
oc get nodes -o wide
oc get clusteroperators
oc get co | awk '$3 != "True" { print }'
```

- Verification (confirm no degraded operators):

```sh
oc get co -o jsonpath='{range .items[*]}{.metadata.name}{" deg="}{.status.conditions[?(@.type=="Degraded")].status}{"\n"}{end}'
```

### Exercise 4 — Create and switch projects

- Task: create two projects, `alpha` and `beta`, with a description and display
  name, then switch between them.
- Expected outcome: `oc get projects` lists both; `oc project` switches the
  active project to `beta`.

```sh
oc new-project alpha --description="Module 1 practice" --display-name="Alpha Practice"
oc new-project beta --description="Module 1 practice" --display-name="Beta Practice"
oc project beta
oc get projects
```

- Verification:

```sh
oc project
oc get project alpha -o jsonpath='{.metadata.annotations.openshift.io/description}{"\n"}'
```

### Exercise 5 — Query with output formats and label selectors

- Task: list all pods in the `openshift-monitoring` namespace with labels, then
  use JSONPath to pull just pod names and their node names.
- Expected outcome: pod names plus a JSONPath summary line listing each pod
  name and node.

```sh
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=prometheus -o wide
oc get pods -n openshift-monitoring -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.nodeName}{"\n"}{end}'
```

- Verification (non-empty output, at least one pod listed):

```sh
oc get pods -n openshift-monitoring -o name | wc -l
```

## Test yourself

When you can do Exercises 1–5 without the book, sit a simulator attempt:

- **Domain**: `cluster-configuration` in the `ex280-admin` bank.
- **First pass**: **Training** mode — read the explanation for every question,
  right or wrong.
- **Second pass**: **Mastery** mode, focused on the same domain, timed.
- Aim for 80%+ on the Mastery pass before starting Module 2; below that, redo
  Exercises 3–5, which carry most of the mechanics.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Admin path index](README.md) — next: Module 2, Projects, RBAC, users and
  ServiceAccounts.
