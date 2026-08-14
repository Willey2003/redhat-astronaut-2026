# Admin Module 3 — Security Context Constraints, pod security, quotas and limits

Module 2 decided *who* may act. This module decides *what workloads are allowed
to do* once they run, and *how many resources a project may burn*. SCCs are
uniquely OpenShift and a high-yield exam topic; quotas and limits are the
resource-governance half of the same module.

## Learning objectives

1. Explain what a SecurityContextConstraints (SCC) is and how it gates pod
   creation at admission time.
2. Distinguish the default SCCs and grant them to users, groups, and
   ServiceAccounts.
3. Configure the pod `securityContext` and understand how OpenShift maps SCCs
   onto Pod Security Admission labels.
4. Enforce resource budgets with ResourceQuota and per-pod defaults with
   LimitRange.

## Key concepts

- **SCC = admission-time gate**: every pod is validated and, when needed,
  mutated by the SCC admission controller. A pod must satisfy an SCC that the
  requesting identity may "use"; otherwise creation is rejected.
- **Default SCCs**: `restricted-v2` (default for new projects, most locked
  down), `restricted`, `nonroot`, `anyuid`, `hostaccess`, `hostmount-anyuid`,
  `hostnetwork`, and `privileged`. They differ mainly in allowed UIDs,
  privileged escalation, host namespaces, and volume hostPath access.
- **`use` verb**: SCCs are granted with the `use` verb, not `create`/`update`.
  The convenience commands are `oc adm policy add-scc-to-user <scc> <user>`,
  `... -to-group <group>`, and `... -to-serviceaccount <sa> -n <ns>`.
- **Identity matters**: which SCCs apply depends on who/what *creates* the pod
  — a Deployment created by a ServiceAccount is checked against that SA's SCC
  grants, not the human operator's.
- **Pod securityContext fields**: `runAsUser`, `runAsNonRoot`,
  `runAsGroup`, `seLinuxOptions`, `allowPrivilegeEscalation`, `capabilities`,
  `fsGroup`, `readOnlyRootFilesystem`. The SCC is the policy; these are the
  requests.
- **Mutating admission**: when you create a pod that requests nothing, the SCC
  layer *defaults* fields (for example forcing `runAsNonRoot=true` under
  `restricted-v2`). Expect `oc get pod -o yaml` to show values you never
  wrote.
- **Pod Security Admission (PSA)**: OpenShift 4.x runs PSA alongside SCCs.
  Namespaces carry `pod-security.kubernetes.io/enforce`/`audit`/`warn` labels
  with `privileged`, `baseline`, or `restricted` profiles; OpenShift keeps the
  SCC and PSA layers in sync, and violations appear in namespace events.
- **ResourceQuota**: a namespaced object capping aggregate usage — CPU, memory,
  storage, and *counts* of resources (`count/pods`,
  `count/deployments.apps`). Requests are enforced at admission; the workload
  is rejected if the quota would be exceeded.
- **LimitRange**: sets per-pod and per-container defaults and limits
  (`default`, `defaultRequest`, `max`, `min`) in a namespace; pods whose
  requests/limits fall outside are rejected or defaulted.
- **Defaults interplay**: LimitRange fills gaps, ResourceQuota caps totals.
  A deployment can fail with quota while individual pods are fine — read the
  events.
- **Inspection**: `oc get scc`, `oc describe scc restricted-v2`, `oc get
  resourcequota`, `oc describe quota`, `oc get limitrange`, and `oc get events
  -n <ns> --sort-by=.lastTimestamp`.
- **Usage metrics**: `oc adm top node` and `oc adm top pod` show live usage,
  useful for sizing quotas.

## Hands-on exercises

### Exercise 1 — A pod that asks too much is rejected

- Task: in project `alpha`, try to create a Deployment that requests
  `privileged` escalation, and observe the SCC admission rejection.
- Expected outcome: pod creation fails; a `FailedCreate` event explains the
  pod does not satisfy any SCC available to the default SA.

```sh
oc project alpha
oc create deployment badpriv --image=gcr.io/google-samples/hello-app:1.0
oc set security-context deployment/badpriv --privileged=true
```

- Verification:

```sh
oc get events -n alpha --sort-by=.lastTimestamp | grep -i scc
oc get pods -n alpha | grep badpriv
```

### Exercise 2 — Grant `anyuid` and prove the UID changes

- Task: create SA `anyuid-runner`, grant it the `anyuid` SCC, and run a pod
  under it; confirm it runs as a UID that `restricted-v2` would have blocked.
- Expected outcome: the pod starts (under `restricted-v2` a runAsUser of 0
  would be rejected), and `oc rsh ... id` shows uid=0.

```sh
oc create sa anyuid-runner -n alpha
oc adm policy add-scc-to-user anyuid -z anyuid-runner -n alpha
oc create deployment anyuid-app --image=gcr.io/google-samples/hello-app:1.0
oc set serviceaccount deployment/anyuid-app anyuid-runner
```

- Verification:

```sh
oc get scc anyuid
oc get pod -n alpha -l app=anyuid-app -o jsonpath='{.items[0].spec.containers[0].securityContext.runAsUser}{"\n"}'
```

### Exercise 3 — Confirm the default SCC on a normal pod

- Task: create a plain Deployment and inspect which SCC it satisfied and what
  the SCC layer defaulted.
- Expected outcome: the pod's `securityContext.runAsNonRoot` is true, and its
  service account's SCC is `restricted-v2` (visible in pod annotations).

```sh
oc create deployment plainapp --image=gcr.io/google-samples/hello-app:1.0
oc get pod -n alpha -l app=plainapp -o yaml | grep -E "runAsNonRoot|openshift.io/scc"
```

- Verification:

```sh
oc get pod -n alpha -l app=plainapp -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'
oc get scc restricted-v2 -o jsonpath='{.allowedCapabilities}{"\n"}'
```

### Exercise 4 — Enforce a ResourceQuota

- Task: create quota `alpha-quota` capping 1 CPU / 512 MiB memory and 3 pods in
  `alpha`, then scale a deployment past it and watch enforcement.
- Expected outcome: scaling to 5 replicas succeeds for the Deployment object
  but pods 4 and 5 are never created; quota events report the failure.

```sh
oc create quota alpha-quota -n alpha --hard=cpu=1,memory=512Mi,pods=3
oc scale deployment plainapp --replicas=5 -n alpha
```

- Verification:

```sh
oc get resourcequota -n alpha
oc get pods -n alpha | wc -l
oc describe quota alpha-quota -n alpha
oc get events -n alpha --sort-by=.lastTimestamp | grep -i quota
```

### Exercise 5 — Apply a LimitRange and observe defaults

- Task: create a LimitRange in `beta` that defaults containers to
  100m CPU / 128 Mi memory, then deploy and confirm the defaults were applied.
- Expected outcome: the running pod's container shows those request/limit
  values even though the Deployment never specified them.

```sh
oc project beta
oc apply -f - <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: beta-limits
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 200m
        memory: 256Mi
EOF
oc create deployment lr-app --image=gcr.io/google-samples/hello-app:1.0
```

- Verification:

```sh
oc get limitrange -n beta
oc get pod -n beta -l app=lr-app -o jsonpath='{.items[0].spec.containers[0].resources}{"\n"}'
```

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `authentication-security` in the `ex280-admin` bank (the SCC and
  pod-security questions live here alongside RBAC).
- **First pass**: **Training** mode — study the explanations, especially for
  any question about *which* SCC maps to a given pod request.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: also re-run a focused Mastery on `cluster-configuration` and the
  RBAC half of `authentication-security`; SCC admission builds directly on the
  binding mechanics from Module 2.

Rejection messages are your study guide: whenever a pod is refused, read the
event, decide which SCC is missing, and redo Exercise 2's grant workflow.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Admin path index](README.md) — next: Module 4, Networking — Services,
  Routes, NetworkPolicies.
