# Admin Module 2 — Projects, RBAC, users and ServiceAccounts

Access control is the most exam-relevant admin skill after raw CLI fluency.
This module covers the identity side (who), the authorization side (what they
may do), and ServiceAccounts (what workloads may do). Module 3 then hardens
*how* workloads run.

## Learning objectives

1. Create, configure, and manage projects as annotated namespaces.
2. Model authorization with Roles, ClusterRoles, and their bindings.
3. Map identity-provider users onto RBAC grants and verify access with
   `oc auth can-i`.
4. Create ServiceAccounts, grant them roles, and obtain their tokens.

## Key concepts

- **Project = namespace + metadata**: projects carry annotations such as
  `openshift.io/description`, `openshift.io/display-name`, and
  `openshift.io/requester`. `oc new-project` sets these; `oc get projects`
  shows only the projects your identity can see.
- **RBAC objects**: a **Role** is a namespaced set of rules; a **ClusterRole**
  is cluster-scoped but can be bound inside a namespace. **RoleBinding** binds
  a Role to a user/group/ServiceAccount in a namespace; **ClusterRoleBinding**
  binds a ClusterRole cluster-wide.
- **Verbs and resources**: rules combine resources (pods, services,
  deployments) with verbs (get, list, watch, create, update, patch, delete).
  "Use" is the verb for granting special resources like SCCs (Module 3).
- **Built-in ClusterRoles**: `view`, `edit`, `admin`, and `cluster-admin`
  form the escalation ladder. `admin` adds project-level rights like
  `Roles/RoleBindings`; `cluster-admin` is the only truly unrestricted role.
- **Granting**: `oc policy add-role-to-user <role> <user> -n <ns>` is the
  convenience command; it is shorthand for creating a RoleBinding.
- **Verifying access**: `oc auth can-i <verb> <resource> -n <ns>` asks the
  authorization layer directly; `oc auth can-i --list -n <ns>` dumps every
  permission you hold there.
- **Identity providers**: OpenShift authenticates via configured IDPs
  (HTPasswd, LDAP, Keystone, OpenID Connect...). Users are identities mapped
  into the cluster; on a local cluster the practical bootstraps are the
  `kubeadmin` user and HTPasswd IDP users. There is no `oc create user`.
- **ServiceAccounts**: a ServiceAccount is the workload's identity — the
  principal that pods run as. Every namespace gets a `default` SA
  automatically; `oc create sa <name>` adds more, and `oc describe sa <name>`
  shows bound roles and associated secrets.
- **SA tokens**: modern OpenShift issues short-lived tokens on demand via the
  TokenRequest API: `oc create token <sa>`. Mounted service-account tokens are
  injected automatically into pods (`/var/run/secrets/kubernetes.io/...`).
- **RBAC aggregation**: aggregated ClusterRoles like `edit` pull in rules from
  sub-roles (`system:aggregate-to-edit`) — a pattern you will also see in
  `system:aggregate-to-view` and `system:aggregate-to-admin`.
- **Groups**: roles can bind to groups, which IDPs populate — this keeps
  bindings small when users change.

## Hands-on exercises

### Exercise 1 — Create and annotate projects

- Task: create project `alpha` with a display name and description, then check
  the annotations that OpenShift added for you.
- Expected outcome: `alpha` appears in `oc get projects`, with the custom
  annotations you set.

```sh
oc new-project alpha --description="Module 2 practice" --display-name="Alpha Lab"
oc annotate project alpha openshift.io/requester=student
```

- Verification:

```sh
oc get project alpha -o jsonpath='{.metadata.annotations.openshift.io/description}{"\n"}'
oc get project alpha -o jsonpath='{.metadata.annotations.openshift.io/requester}{"\n"}'
```

### Exercise 2 — Grant and verify a view role

- Task: give a (possibly non-existent) user `viewer-alpha` the `view`
  ClusterRole inside `alpha`, then verify what that grants.
- Expected outcome: `oc auth can-i list pods -n alpha --as viewer-alpha`
  returns `yes`, and `delete deployments` returns `no`.

```sh
oc policy add-role-to-user view viewer-alpha -n alpha
oc policy can-i list pods -n alpha --as viewer-alpha
```

- Verification:

```sh
oc auth can-i list pods -n alpha --as viewer-alpha
oc auth can-i delete deployments -n alpha --as viewer-alpha
```

### Exercise 3 — Create a ServiceAccount with edit rights

- Task: create SA `builder` in `alpha`, bind `edit` to it, and confirm the
  SA can create deployments but not bind roles.
- Expected outcome: `--as=system:serviceaccount:alpha:builder` can create
  deployments (yes) but cannot create rolebindings (no).

```sh
oc create sa builder -n alpha
oc policy add-role-to-user edit system:serviceaccount:alpha:builder -n alpha
oc auth can-i create deployments -n alpha --as=system:serviceaccount:alpha:builder
oc auth can-i create rolebindings -n alpha --as=system:serviceaccount:alpha:builder
```

- Verification:

```sh
oc get sa -n alpha
oc describe sa builder -n alpha
```

### Exercise 4 — Request a ServiceAccount token

- Task: mint a token for SA `builder` and inspect it.
- Expected outcome: `oc create token builder -n alpha` prints a long JWT with
  three dot-separated segments; it is not stored as a persistent secret.

```sh
oc create token builder -n alpha
oc create token builder -n alpha --duration=1h
```

- Verification (a well-formed token decodes to three parts):

```sh
oc create token builder -n alpha | awk -F. '{ print NF }'
```

### Exercise 5 — Inspect bindings and RBAC aggregation

- Task: inspect what `edit` actually grants and list all RoleBindings in
  `alpha`.
- Expected outcome: `oc describe clusterrole edit` shows the
  `system:aggregate-to-edit` aggregation rule, and `oc get rolebindings`
  lists the binding you created in Exercise 3.

```sh
oc describe clusterrole edit
oc get rolebindings -n alpha
oc describe rolebinding edit -n alpha
```

- Verification:

```sh
oc get clusterrole edit -o jsonpath='{.aggregationRule.clusterRoleSelectors[0].matchLabels}{"\n"}'
oc auth can-i --list -n alpha --as=system:serviceaccount:alpha:builder | grep -i deployment
```

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `authentication-security` in the `ex280-admin` bank.
- **First pass**: **Training** mode, focused on that domain — study the
  explanations.
- **Second pass**: **Mastery** mode, timed, same domain.
- **Also**: sit a focused Mastery attempt on `cluster-configuration` (Module 1)
  to keep it fresh — that is a deliberate spiral.

If `oc auth can-i` answers surprise you, re-read the concept bullets on
bindings and aggregation, then redo Exercises 2–3.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Admin path index](README.md) — next: Module 3, Security Context
  Constraints, pod security, quotas and limits.
