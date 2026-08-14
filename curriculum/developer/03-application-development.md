# Developer Module 3 — Routing and services for apps: Routes, ConfigMaps and Secrets

Now that applications exist, make them reachable and configurable. This module
covers the Route types a developer must know (including TLS modes), advanced
routing (path-based, weighted/canary), and the two injection mechanisms —
ConfigMaps and Secrets — for getting configuration into a workload without
rebuilding its image.

## Learning objectives

1. Expose applications with Routes in edge, reencrypt, and passthrough modes
   and verify them with curl.
2. Route traffic by path and split it across backends by weight for
   blue/green and canary delivery.
3. Create ConfigMaps and Secrets from literals and files and inject them as
   environment variables and volumes.
4. Consume TLS secrets and registry pull secrets, and keep configuration
   updates flowing without redeploys where the platform allows.

## Key concepts

- **Route**: the OpenShift external-exposure object. A Route fronts a Service
   on a hostname handled by the Ingress Controller (HAProxy in
   `openshift-ingress`). `oc expose svc/<name>` is the shortcut; `oc create
   route <edge|reencrypt|passthrough> <name> --service=<svc>` is explicit.
- **TLS modes**: **edge** — router terminates TLS, plain HTTP to pods (you
   provide/proxy certs at the router); **reencrypt** — router terminates TLS
   and re-encrypts to the pods, so pods must serve TLS and the Route carries a
   destination CA; **passthrough** — the TLS stream passes through unmodified
   to the pod, so the *pod* terminates it. `insecureEdgeTerminationPolicy`:
   `None`/`Redirect`/`Allow` decides plain-HTTP handling.
- **Serving certificates**: annotate a Service with
   `service.beta.openshift.io/serving-cert-secret-name=<secret>` to auto-issue
   a signed serving cert — the clean way to give pods TLS for reencrypt Routes.
- **Path-based routing**: several Routes may share a hostname if `spec.path`
   differs; the longest matching path prefix wins. One host, many paths.
- **Weighted routing**: one Route, multiple backends. `oc set route-backends
   <route> <svc1>=80 <svc2>=20` (or `oc set weight`) splits traffic — the
   canary mechanism.
- **ConfigMap**: key/value or file payload (`oc create cm <name> --from-literal=k=v
   --from-file=...`). Injected via `env`, `envFrom`, or a volume mount. Built
   into the pod spec; updates to the ConfigMap are *not* copied into running
   containers automatically (except mounted files, which kubelet refreshes).
- **Secret**: like a ConfigMap but base64-encoded and with secret-specific
   kinds — `opaque`, `tls` (`oc create secret tls <name> --cert --key`),
   `dockerconfigjson` (image pull credentials, `oc create secret
   docker-registry`). Mounted as files or env vars; stored in etcd, so avoid
   logging them.
- **Injection mechanics**: `oc set env deploy/<name> --from=secret/<name>` and
   `oc set env deploy/<name> --from=configmap/<name>` for envs;
   `oc set volume deploy/<name> --add --name=cfg --type=configmap
   --configmap-name=<cm> --mount-path=/etc/cfg` and `--type=secret
   --secret-name=<s>` for files.
- **Image pull secrets**: a pod needs `imagePullSecrets` (or the SA must carry
   them) to pull from a private registry. `oc create secret docker-registry
   puller --docker-server=... --docker-username=... --docker-password=...` then
   attach to the SA: `oc secrets link sa/default puller --for=pull`.
- **Env vs mounted**: env values are snapshotted at pod creation (change =
   redeploy); mounted files update live and can be watched by the app.
- **Verification**: `curl -k` for edge/reencrypt, plain `curl` for insecure,
   and `curl -v` to see which backend answered in a weighted split.

## Hands-on exercises

### Exercise 1 — Edge Route and HTTPS verification

- Task: in project `router`, deploy `web` (hello-app), expose it as a Service,
   then create an edge Route and verify over HTTPS.
- Expected outcome: `oc get route` shows `edge`, and `curl -k -w '%{http_code}'`
   returns 200.

```sh
oc new-project router
oc create deployment web --image=gcr.io/google-samples/hello-app:1.0
oc expose deployment web --port=8080 --target-port=8080 --name=web-svc
oc create route edge web-edge --service=web-svc
```

- Verification:

```sh
oc get route web-edge -o jsonpath='{.spec.tls.termination}{"\n"}'
curl -k -s -o /dev/null -w '%{http_code}\n' https://$(oc get route web-edge -o jsonpath='{.spec.host}')
```

### Exercise 2 — Reencrypt and passthrough Routes

- Task: give `web-svc` a serving certificate, create a reencrypt Route using
   it, and create a passthrough Route for a second, TLS-serving deployment.
- Expected outcome: `oc get route` shows `reencrypt` (with destination CA from
   the serving-cert secret) and `passthrough` terminations.

```sh
oc annotate service web-svc service.beta.openshift.io/serving-cert-secret-name=web-cert
oc get configmap service-ca.crt -o jsonpath='{.data.service-ca\.crt}' > /tmp/web-ca.crt
oc create route reencrypt web-reencrypt --service=web-svc --dest-ca-cert=/tmp/web-ca.crt
oc create deployment tlsapp --image=nginx:1.25
oc create route passthrough tls-route --service=tlsapp
```

- Verification:

```sh
oc get secret web-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | head -1
oc get route web-reencrypt -o jsonpath='{.spec.tls.termination}'
oc get route web-reencrypt -o jsonpath='{.spec.tls.destinationCACertificate}' | head -c 40; echo
oc get route tls-route -o jsonpath='{.spec.tls.termination}{"\n"}'
```

> The reencrypt exercise verifies the Route *shape* (termination mode +
   destination CA wired to the serving-cert secret). A working end-to-end
   reencrypt flow needs the pod to actually serve TLS; on a plain nginx that
   means configuring the serving cert into the pod — do that as a stretch goal
   by mounting `web-cert` and pointing nginx's TLS at it.

### Exercise 3 — Path-based routing

- Task: on one hostname serve `/blue` from `web-svc` and `/green` from a second
   service, sharing the host.
- Expected outcome: two Routes share `hostname`, one with `path=/blue`, one with
   `path=/green`, and each curl path reaches the right app.

```sh
oc create deployment web2 --image=gcr.io/google-samples/hello-app:2.0
oc expose deployment web2 --port=8080 --target-port=8080 --name=web2-svc
oc create route edge blue-route --service=web-svc --path=/blue --hostname=split.apps-crc.testing
oc create route edge green-route --service=web2-svc --path=/green --hostname=split.apps-crc.testing
```

- Verification:

```sh
oc get routes -o custom-columns=NAME:.metadata.name,HOST:.spec.host,PATH:.spec.path,TO:.spec.to.name
curl -k -s https://split.apps-crc.testing/blue -o /dev/null -w '%{http_code}\n'
curl -k -s https://split.apps-crc.testing/green -o /dev/null -w '%{http_code}\n'
```

### Exercise 4 — Weighted (canary) routing

- Task: on `web-edge`, split traffic 80/20 between `web-svc` and `web2-svc`.
- Expected outcome: `oc get route` (or `oc set route-backends --help`) shows
   weights 80 and 20 on the two services; repeated curls hit both backends.

```sh
oc set route-backends web-edge web-svc=80 web2-svc=20
```

- Verification:

```sh
oc get route web-edge -o jsonpath='{range .spec.alternateBackends[*]}{.name}{" weight="}{.weight}{"\n"}{end}'
for i in $(seq 1 20); do curl -k -s https://$(oc get route web-edge -o jsonpath='{.spec.host}'); echo; done | grep -c "Version"
```

> hello-app v1.0 answers "Hello, world!" and v2.0 adds "Version: 2.0.0", so the
> `grep -c "Version"` line counts v2 answers — expect roughly 4 of 20 with an
> 80/20 split.

### Exercise 5 — ConfigMaps and Secrets injection

- Task: in project `cfg`, create ConfigMap `app-config` (a greeting key) and a
   TLS secret `app-tls`, inject the ConfigMap key as an env var, mount the
   ConfigMap as files, and verify both from inside a pod.
- Expected outcome: the pod's env var holds the ConfigMap value, and the
   mounted file `/etc/cfg/greeting.txt` contains it.

```sh
oc new-project cfg
oc create cm app-config --from-literal=greeting=hello-configmap --from-file=greeting.txt=<(echo "from-file")
oc create secret tls app-tls --cert=/etc/pki/tls/certs/ca-bundle.crt --key=/etc/pki/tls/certs/localhost.key 2>/dev/null || true
oc create deployment cfgapp --image=registry.access.redhat.com/ubi9/ubi-minimal:latest -- /bin/sh -c 'sleep 3600'
oc set env deployment/cfgapp --from=configmap/app-config
oc set volume deployment/cfgapp --add --name=cfgvol --type=configmap --configmap-name=app-config --mount-path=/etc/cfg
oc set volume deployment/cfgapp --add --name=secretvol --type=secret --secret-name=app-tls --mount-path=/etc/tls
```

- Verification:

```sh
oc exec deployment/cfgapp -- env | grep greeting
oc exec deployment/cfgapp -- cat /etc/cfg/greeting.txt
oc exec deployment/cfgapp -- ls -la /etc/tls
```

> The TLS secret creation may fail if `localhost.key` is absent on your host —
   that is fine: the graded mechanics are `oc create secret tls ... --cert
   ... --key ...` and mounting it. If it fails, create an opaque secret instead
   and mount that; the point is the inject-and-read flow.

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `application-routing` in the `ex288-developer` bank.
- **First pass**: **Training** mode — study explanations on TLS termination
   modes and the env-vs-mounted-file difference.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: re-sit `application-deployment` Mastery (Module 1) — canary
   routing depends on those Deployment skills.

If TLS modes blur together, make a one-line crib: **edge = TLS at router only,
reencrypt = router TLS then TLS to pod, passthrough = pod TLS end to end** —
then redo Exercises 1–2.

## See also

- Red Hat EX288 exam page: redhat.com/training/ex288
- [Developer path index](README.md) — next: Module 4, GitOps and CI/CD —
   Pipelines, Argo CD, helm.
