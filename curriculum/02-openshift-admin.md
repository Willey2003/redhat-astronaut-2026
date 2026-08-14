# Module 2 — OpenShift administration (EX280 · EX229)

Module 2 turns you from someone who runs containers into someone who runs the
**platform**. EX280 is the core administration certification: architecture,
identity and security, networking, storage, and Operators. EX229 is the
specialist certification for managing containers on the command line, which
here is folded into the "cluster administration" half — the `oc`-centric view
of everything Module 1 did with `podman`. A running cluster is required from
now on; use **Red Hat OpenShift Local (CRC)** or OKD, 4+ vCPU and 16+ GiB RAM.

## Learning objectives

1. Explain the OpenShift 4 control plane and how Cluster Operators own the
   platform, and check cluster health with `oc`.
2. Manage projects, users, ServiceAccounts, RBAC roles and bindings, and
   `oc adm` administrative verbs.
3. Configure authentication identity providers, quotas, limits, and pod
   security (SCCs) to control what can run.
4. Expose and secure workloads: Services, Routes with TLS, NetworkPolicies,
   and external access.
5. Provision persistent storage: StorageClasses, PersistentVolumeClaims, and
   the lifecycle of PV/PVC/Routes.
6. Install, configure, and use Operators and their operand instances.

## Key concepts

- **Control plane**: the API server is the front door; etcd stores state; the
  scheduler places pods; controllers reconcile desired state. `oc get co`
  lists Cluster Operators and their `Available`/`Degraded` conditions —
  check `Degraded=False` everywhere.
- **oc vs kubectl**: `oc` is a superset. Prefer it everywhere — it also
  speaks projects, routes, buildconfigs, and scc. `oc whoami`, `oc project`,
  `oc config get-contexts`.
- **Projects and RBAC**: a project is a namespace plus annotations.
  `oc new-project alpha`, `oc adm policy add-role-to-user view alice -n
  alpha`, `oc create clusterrolebinding`. Roles (`view`, `edit`, `admin`,
  `cluster-admin`) map to verbs on resources.
- **ServiceAccounts**: identities for pods and automation. `oc create sa
  builder`, `oc policy add-role-to-user admin -z builder -n alpha` (the `-z`
  targets a ServiceAccount). Tokens let pods call the API.
- **Authentication**: `oc login -u <user> -p <pw>`; identity providers are
  configured in the `OAuth` CR — `htpasswd`, LDAP, OIDC, etc. `oc create
  secret` + `oc create -f oauth.yaml` wires an htpasswd identity provider for
  the lab.
- **SCCs (SecurityContextConstraints)**: gate what a pod may do — run as
  root, privileged, hostNetwork, SELinux context. `oc get scc`,
  `oc describe scc restricted`, `oc adm policy add-scc-to-user anyuid -z
  default -n alpha`. OpenShift ships `restricted`, `anyuid`, `privileged`, and
  more.
- **Quotas and limits**: `oc create quota q1 -n alpha --hard=pods=10`,
  `oc create limitrange limits -n alpha --max=cpu=2,memory=2Gi`. Quota caps
  totals; LimitRange caps individual pods.
- **Services and Routes**: Service = stable ClusterIP. A Route exposes a
  Service over HTTP(S). `oc expose service web --port=8080 --name=web-route
  --hostname=web.apps.example.com`, TLS with `--terminate edge`. Route types:
  `edge` (TLS on router), `passthrough` (end-to-end TLS), `reencrypt`.
- **NetworkPolicy**: `kind: NetworkPolicy` with `podSelector`,
  `ingress`/`egress` rules and `ipBlock`/`namespaceSelector` peers. OpenShift
  uses OVN-Kubernetes; policies are namespace-scoped.
- **Storage**: a StorageClass names a CSI driver; a PVC requests storage; the
  PV is dynamically provisioned. `oc get storageclass`, `oc get pvc`. Reclaim
  policies (`Delete`/`Retain`) decide what happens to the PV when the PVC
  goes.
- **Operators**: install via OperatorHub. `oc get packagemanifests | grep
  -i postgres`, `oc create subscription ... -n openshift-operators`, then a
  `ClusterServiceVersion` (CSV) and custom resources (CRs) for operands.
- **oc adm essentials**: `oc adm drain node1`, `oc adm cordon node1`,
  `oc adm top node`, `oc adm policy`, `oc adm upgrade` (via ClusterVersion).
  These are the "administrative" verbs EX280 loves.

## Hands-on exercises

All exercises assume you are `kubeadmin` on your local cluster
(`oc login -u kubeadmin -p <pw> https://api.crc.testing:6443`).

### Exercise 1 — Verify cluster health and identity

- Task: confirm who you are, the cluster version, and every Cluster
  Operator's condition.
- Expected outcome: `oc whoami` prints `kubeadmin`; `oc get co` shows all
  operators `Available=True` and `Degraded=False`.

```sh
oc whoami
oc version
oc get clusterversion
oc get co | awk '$3 != "True" { print }'
oc get nodes -o wide
```

- Verification (no degraded operators, node Ready):

```sh
oc get co -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.conditions[?(@.type=="Degraded")].status}{"\n"}{end}' | grep -v False
oc get nodes | grep Ready
```

### Exercise 2 — RBAC: users, roles, and bindings

- Task: create an htpasswd user, a project, and grant that user the `admin`
  role in it; verify with `oc auth can-i`.
- Expected outcome: user `alice` can list deployments and create pods in
  project `alpha`, but not delete nodes.

```sh
htpasswd -c -B /tmp/htpasswd alice
oc create secret generic htpass-secret --from-file=htpasswd=/tmp/htpasswd -n openshift-config
# (then patch the OAuth CR to reference it — see the lab bank's explanation)
oc new-project alpha
oc adm policy add-role-to-user admin alice -n alpha
oc login -u alice -p <pw> https://api.crc.testing:6443
oc auth can-i create deployments -n alpha
oc auth can-i delete nodes
```

- Verification (first can-i says yes, second no):

```sh
oc auth can-i create deployments -n alpha
oc auth can-i delete nodes
```

### Exercise 3 — SCC and pod security

- Task: deploy a pod that needs to run as UID 0, then grant the `anyuid` SCC
  to the project's default ServiceAccount so it starts.
- Expected outcome: before the grant the pod stays `CreateContainerConfigError`
  (or is rejected); after `add-scc-to-user` it reaches `Running`.

```sh
oc new-project lab3
oc create deployment db --image=postgres:15 --replicas=1
oc get pods -n lab3
oc adm policy add-scc-to-user anyuid -z default -n lab3
oc rollout restart deployment/db -n lab3
oc get pods -n lab3
```

- Verification (pod Running after the grant):

```sh
oc get pods -n lab3 -l app=db
```

### Exercise 4 — Routes and NetworkPolicy

- Task: deploy `web`, expose a Service, create an edge TLS Route, then write a
  NetworkPolicy that only allows traffic from pods labelled `app=front`.
- Expected outcome: the Route answers HTTPS; a probe pod labelled `app=front`
  can reach the web Service while an unlabelled pod cannot.

```sh
oc new-project lab4
oc create deployment web --image=nginx:1.24
oc expose deployment web --port=80
oc create route edge web-route --service=web
curl -skI https://$(oc get route web-route -o jsonpath='{.spec.host}') | head -1
oc apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-front
  namespace: lab4
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: front
      ports:
        - protocol: TCP
          port: 80
EOF
oc run probe --image=curlimages/curl --rm -it --restart=Never -l app=front -- curl -sI http://web.lab4.svc:80
```

- Verification (front pod gets HTTP 200, HTTPS route answers):

```sh
oc run probe2 --image=curlimages/curl --rm -it --restart=Never -- curl -sI http://web.lab4.svc:80   # expect timeout
```

### Exercise 5 — Storage and Operators

- Task: create a StorageClass-backed PVC, mount it into a Deployment, and
  verify data survives a pod recreation. Then install an Operator from
  OperatorHub and create its CR.
- Expected outcome: data written to `/mnt/data` survives; the Operator's CSV
  reaches `Succeeded` and the CR creates the operand.

```sh
oc new-project lab5
oc apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: lab5
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
EOF
oc set volumes deployment/db --add --name=data -t pvc --claim-name=data -m /mnt/data
oc get pvc -n lab5
oc get packagemanifests -n openshift-marketplace | grep -i postgres
```

- Verification (PVC Bound; volume mounted):

```sh
oc get pvc data -n lab5
oc get deployment db -n lab5 -o jsonpath='{.spec.template.spec.volumes[0].persistentVolumeClaim.claimName}{"\n"}'
```

## Test yourself

When you can do Exercises 1–5 without the book, sit simulator attempts:

- **Bank**: `banks/ex280-admin`
- **First pass**: **Training** mode, `focus_domain = authentication-security`
  — read every explanation, right or wrong.
- **Second pass**: **Mastery** mode on `networking` and `storage`, timed.
- **Third pass**: **Training** on `banks/ex229` → `cluster-administration` and
  `security-rbac` to pick up the container-management perspective.
- Aim for 80%+ on the Mastery passes before starting Module 3; below that,
  redo Exercises 2–4, which carry the exam's most-tested mechanics.

## Self-check quiz

1. **Q**: What does `oc adm policy add-scc-to-user anyuid -z default -n alpha`
   do? **A**: It adds the `anyuid` SCC to the `default` ServiceAccount in
   project `alpha`, letting pods there run containers as any user ID.
2. **Q**: How do you check whether a user may delete nodes? **A**: `oc auth
   can-i delete nodes --as <user>` (or from that user's session).
3. **Q**: What is the difference between a Service and a Route? **A**: A
   Service is a cluster-internal stable IP; a Route is an OpenShift object
   that exposes a Service externally over HTTP/HTTPS through the router.
4. **Q**: What happens to a PV when its PVC is deleted and the StorageClass
   reclaim policy is `Delete`? **A**: The PV is deleted with its underlying
   storage. Use `Retain` to keep the volume for manual recovery.
5. **Q**: Why would a pod fail with `CreateContainerConfigError`? **A**: A
   ConfigMap, Secret, or SCC/security context issue at pod admission — check
   `oc describe pod` and the SCC the pod is bound to.

## See also

- EX280 and EX229 certification pages on redhat.com (referenced by name).
- OpenShift 4 product documentation (docs.openshift.com) — RBAC, SCCs, Routes,
  NetworkPolicy, Operators chapters.
- `oc explain route`, `oc explain scc`, `oc explain networkpolicy`.
- [Module 3 — OpenShift application development (EX282 · EX480)](03-openshift-developer.md)
  — next: the developer side of the platform you now administer.
