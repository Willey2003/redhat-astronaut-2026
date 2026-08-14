# Admin Module 4 — Networking: Services, Routes, NetworkPolicies

This module covers how traffic reaches pods: Services for east-west and
cluster-internal addressing, Routes for north-south exposure (a uniquely
OpenShift resource), and NetworkPolicies for isolating east-west traffic.
Module 5 then provisions the storage those pods can actually use.

## Learning objectives

1. Create ClusterIP, NodePort, and LoadBalancer Services and consume them via
   cluster DNS.
2. Expose applications with Routes in edge, reencrypt, and passthrough TLS
   modes.
3. Route traffic by hostname and path, and split traffic across services by
   weight.
4. Write NetworkPolicies to default-deny and selectively allow pod traffic.

## Key concepts

- **Service = stable VIP**: a Service selects pods by labels and load-balances
  to their endpoints. The DNS name inside the cluster is
  `<service>.<namespace>.svc.cluster.local`.
- **Service types**: `ClusterIP` (default, internal only), `NodePort` (same
  port on every node, `oc get svc` shows the port), `LoadBalancer` (external
  LB provisioning; on OpenShift Local this resolves to a `*.apps-crc.testing`
  hostname via the router).
- **Endpoints**: the concrete pod IP:port pairs behind a Service
  (`oc get endpoints`). A selector mismatch leaves the Service with no
  endpoints — the most common "why is it down?" cause.
- **Route**: the OpenShift north-south object. A Route fronts a Service on an
  external hostname (`<name>-<namespace>.<router-domain>`); the default Ingress
  Controller (in `openshift-ingress`, HAProxy-based) serves it. `oc expose svc
  <svc>` is the shortcut.
- **TLS termination modes**: **edge** (router terminates TLS, sends plain HTTP
  to pods), **reencrypt** (router terminates then *re-encrypts* to the pods'
  TLS), **passthrough** (router passes the TLS stream untouched to the pod).
  `insecureEdgeTerminationPolicy` decides what happens to plain HTTP hits:
  `None`, `Redirect`, or `Allow`.
- **Host-based vs path-based**: a Route's `spec.host` matches the Host header;
  multiple Routes may share a hostname as long as their `spec.path` differ
  (longest path prefix wins). Wildcard hosts (`*.example.com`) are possible.
- **Weighted routing**: a Route can point at one Service with alternate
  backends and per-backend weights — the canary mechanism for blue/green
  rollouts (`oc set route-backends`).
- **Service serving certs**: annotate a Service with
  `service.beta.openshift.io/serving-cert-secret-name=<secret>` to get a
  signed serving certificate — essential for reencrypt Routes to TLS-enabled
  pods.
- **NetworkPolicy**: namespaced, default *allow-all* unless policies exist.
  A policy with `podSelector: {}` (matches everything) changes the default to
  deny. Rules combine `podSelector` (targets), `ingress`/`egress` blocks, and
  peers via `ipBlock`, `namespaceSelector`, or `podSelector`.
- **CNI**: OpenShift 4 uses OVN-Kubernetes by default, which enforces
  NetworkPolicies. Policy take effect live; `oc describe networkpolicy` shows
  rules.
- **DNS debugging**: `oc exec <pod> -- getent hosts <svc>.<ns>.svc.cluster.local`
  and `oc run -it --rm dns --image=... -- /bin/sh` are your probe tools.

## Hands-on exercises

### Exercise 1 — ClusterIP Service and cluster DNS

- Task: in `alpha`, deploy `hello` and expose it as a ClusterIP Service, then
  resolve its DNS from another pod.
- Expected outcome: Service `hello` gets a ClusterIP; a test pod resolves
  `hello.alpha.svc.cluster.local` to that IP.

```sh
oc project alpha
oc create deployment hello --image=gcr.io/google-samples/hello-app:1.0
oc expose deployment hello --port=8080 --target-port=8080
oc run test --image=gcr.io/google-samples/hello-app:1.0 --restart=Never --command -- /bin/sh -c 'sleep 300'
```

- Verification:

```sh
oc get svc hello -o wide
oc exec test -- getent hosts hello.alpha.svc.cluster.local
oc get endpoints hello
```

### Exercise 2 — Edge Route

- Task: expose Service `hello` as an edge-terminated Route on host
  `hello-alpha.apps-crc.testing` and hit it over HTTPS.
- Expected outcome: `curl -k https://hello-alpha.apps-crc.testing` returns the
  hello-app response; `oc get route` shows termination `edge`.

```sh
oc create route edge hello-route --service=hello --hostname=hello-alpha.apps-crc.testing
```

- Verification:

```sh
oc get route hello-route -o jsonpath='{.spec.tls.termination}{"\n"}'
curl -k -s -o /dev/null -w '%{http_code}\n' https://hello-alpha.apps-crc.testing
```

### Exercise 3 — Reencrypt Route with a serving certificate

- Task: give Service `hello` a serving certificate secret, then create a
  reencrypt Route that terminates TLS at the router and re-encrypts to the pod.
- Expected outcome: the Service has a serving-cert secret, the Route has
  `termination: reencrypt` and a `destinationCACertificate`, and the route
  serves traffic (the pod answers plain HTTP on its port, so use `insecure`
  destination verification).

```sh
oc annotate service hello service.beta.openshift.io/serving-cert-secret-name=hello-cert
oc get configmap service-ca.crt -o jsonpath='{.data.service-ca\.crt}' > /tmp/service-ca.crt
oc create route reencrypt hello-reencrypt --service=hello --hostname=reencrypt-alpha.apps-crc.testing --dest-ca-cert=/tmp/service-ca.crt
```

- Verification:

```sh
oc get secret hello-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | head -1
oc get route hello-reencrypt -o jsonpath='{.spec.tls.termination}'
oc get route hello-reencrypt -o jsonpath='{.spec.tls.destinationCACertificate}' | head -c 40; echo
```

> Note: on a truly reencrypted service the pod must serve TLS itself. The
> mechanical part to master for the exam is *creating* the reencrypt Route and
> pointing its destination CA at the serving-cert secret's CA; the local
> hello-app is plain HTTP, so this exercise stops at verifying the Route shape.

### Exercise 4 — Path-based routing

- Task: on one hostname, send `/a` to `hello` and `/b` to a second app
  `hello2`.
- Expected outcome: `curl -k https://split-alpha.apps-crc.testing/a` hits the
  `hello` backend and `/b` hits `hello2`; `oc get routes` shows two Routes
  sharing the host with different paths.

```sh
oc create deployment hello2 --image=gcr.io/google-samples/hello-app:2.0
oc expose deployment hello2 --port=8080 --target-port=8080 --name=hello2-svc
oc create route edge a-route --service=hello --path=/a --hostname=split-alpha.apps-crc.testing
oc create route edge b-route --service=hello2-svc --path=/b --hostname=split-alpha.apps-crc.testing
```

- Verification:

```sh
curl -k -s https://split-alpha.apps-crc.testing/a | grep -o 'Hello World.*'
curl -k -s https://split-alpha.apps-crc.testing/b | grep -o 'Hello World.*'
oc get routes -o custom-columns=HOST:.spec.host,PATH:.spec.path,BACKEND:.spec.to.name
```

### Exercise 5 — NetworkPolicy default-deny plus selective allow

- Task: in a new project `netlab`, create `web` and `db` apps, then add a
  default-deny policy and a policy allowing `web` to reach `db` only on port
  5432.
- Expected outcome: from the `web` pod you can reach `db` on 5432; from an
  unrelated pod you cannot connect at all.

```sh
oc new-project netlab
oc create deployment web --image=gcr.io/google-samples/hello-app:1.0
oc create deployment db --image=registry.access.redhat.com/ubi9/ubi-minimal:latest -- /bin/sh -c 'nc -lk 5432; sleep 3600'
oc expose deployment db --port=5432 --target-port=5432 --name=db-svc
oc apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netlab
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 5432
EOF
oc create deployment probe --image=gcr.io/google-samples/hello-app:1.0 --replicas=0
```

- Verification (from within the cluster):

```sh
oc get networkpolicy -n netlab
oc exec deployment/web -- wget -q -O- --timeout=3 http://db-svc.netlab.svc.cluster.local:5432 ; echo "exit=$?"
oc exec deployment/probe -- wget -q -O- --timeout=3 http://db-svc.netlab.svc.cluster.local:5432 ; echo "exit=$?"
```

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `networking` in the `ex280-admin` bank.
- **First pass**: **Training** mode — pay extra attention to Route TLS
  termination questions and NetworkPolicy selector semantics.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: re-sit `cluster-configuration` Mastery; DNS and route hostnames
  build on the `oc` querying fluency from Module 1.

If path-based or weighted routing feels shaky, redo Exercise 4 and then create
a weighted Route by hand with `oc create route edge weighted --service=hello
--alternate-service=hello2-svc --weight=20` and observe the traffic split.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Admin path index](README.md) — next: Module 5, Storage and Operators.
