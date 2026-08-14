# Developer Module 1 — Application deployment models: Deployments, DeploymentConfigs, scaling, probes

Every developer task on OpenShift ends with a workload running. This module
covers the two workload controllers you will meet, how rollouts and scaling
work, and how to make the platform know your app is healthy.

## Learning objectives

1. Create and update applications with both Deployments and DeploymentConfigs.
2. Drive rollouts: status, history, pause/resume, rollback, and `oc rollout
   latest`.
3. Scale workloads horizontally, manually and with the HorizontalPodAutoscaler.
4. Configure liveness, readiness, and startup probes and read the events that
   reveal probe failures.

## Key concepts

- **Deployment**: the Kubernetes-native controller. Manages a ReplicaSet,
   supports rolling updates, and is the default output of `oc create
   deployment`. Image updates are a "rollout" with a full history.
- **DeploymentConfig (DC)**: the OpenShift-native controller. Adds deployment
   *triggers* (redeploy when an ImageStream tag or its Config changes) and
   built-in rollback (`oc rollout undo`/`oc rollout retry`). Under the hood it
   creates a ReplicationController.
- **Creating workloads**: `oc new-app <image|source>` is the fastest path (it
   wires image stream + deployment/DC + service). `oc create deployment
   <name> --image=...` is the explicit path. `oc new-app
   --as-deployment-config` forces a DC.
- **Rollouts**: `oc rollout status deploy/<name>`, `oc rollout history`, `oc
   rollout undo deploy/<name> --to-revision=N`, `oc rollout pause/resume`,
   `oc rollout latest dc/<name>` (force a DC redeploy). New pods come from a
   new ReplicaSet; old ones drain as the new set scales in.
- **Strategies**: Deployments use `RollingUpdate` (configurable
   `maxSurge`/`maxUnavailable`); DeploymentConfigs add a `Recreate` strategy
   (stop-then-start, for stateful workloads). A failed rollout is a timed-out
   progress, not a crash.
- **Scaling**: `oc scale deploy/<name> --replicas=N`. The
   HorizontalPodAutoscaler reads metrics (CPU/memory) and adjusts replicas:
   `oc autoscale deploy/<name> --min=N --max=M --cpu-percent=P`. Note HPA +
   scale conflict — `oc scale` wins until the HPA next reconciles.
- **Self-healing**: the controller maintains `desired == available`. If a pod
   dies, the ReplicaSet replaces it; `oc get deploy -o wide` shows
   DESIRED/AVAILABLE columns; conditions on the object explain stuck states.
- **Probes**: three kinds — **readiness** (in-service only when ready; failures
   remove the pod from Service endpoints, no restart), **liveness** (failure
   restarts the container), **startup** (protects slow apps from liveness
   restarts during boot). Each has `httpGet`, `exec`, or `tcpSocket` plus
   `initialDelaySeconds`/`periodSeconds`/`timeoutSeconds`/`failureThreshold`.
- **Setting probes**: `oc set probe deploy/<name> --readiness --get-url=http://:8080/healthz`
   or hand-edit with `oc edit`. Verify with `oc describe pod` (probe events)
   and `oc get deploy -o yaml`.
- **Labels and selectors**: deployments match pods by label selector
   (`oc get pods -l app=<name>`). Mismatched selectors are the classic cause of
   "no endpoints".
- **Rollout speed matters on the exam**: `oc rollout status --watch`, `oc wait
   --for=condition=Available`.

## Hands-on exercises

### Exercise 1 — Deploy an app with `oc new-app`

- Task: create project `myapp`, deploy the sample hello app from a public image,
  and confirm a Deployment, pods, and a Service appeared automatically.
- Expected outcome: `oc get deploy,svc,pods` shows one Deployment, one Service,
  and at least one Running pod.

```sh
oc new-project myapp
oc new-app gcr.io/google-samples/hello-app:1.0 --name=hello
```

- Verification:

```sh
oc get deploy hello
oc rollout status deploy/hello --watch
oc get pods -l app=hello
oc get svc hello
```

### Exercise 2 — Create an explicit Deployment and drive a rollout

- Task: create Deployment `frontend` from `nginx:1.25` with 3 replicas, then
  update the image to a new tag and roll the update forward, then back.
- Expected outcome: `oc rollout history` shows two revisions; after `undo` the
  pod image is back to the original tag.

```sh
oc create deployment frontend --image=nginx:1.25
oc scale deployment frontend --replicas=3
oc set image deployment/frontend nginx=nginx:1.27
oc rollout status deployment/frontend --watch
oc rollout history deployment/frontend
oc rollout undo deployment/frontend --to-revision=1
```

- Verification:

```sh
oc get deploy frontend -o wide
oc get pods -l app=frontend
oc get deploy frontend -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
oc rollout history deployment/frontend
```

### Exercise 3 — DeploymentConfig with triggers and rollback

- Task: create project `dcapp`, deploy the same image as a DeploymentConfig,
  push a manual rollout, then force an old revision back.
- Expected outcome: `oc get dc` shows the DC; `oc rollout latest dc/demo`
  creates a new ReplicationController revision, and `oc rollout undo dc/demo`
  reverts.

```sh
oc new-project dcapp
oc new-app --as-deployment-config gcr.io/google-samples/hello-app:2.0 --name=demo
oc rollout latest dc/demo
oc rollout history dc/demo
oc rollout undo dc/demo
```

- Verification:

```sh
oc get dc,rc -n dcapp
oc rollout status dc/demo --watch
oc rollout history dc/demo
```

### Exercise 4 — Probes: readiness, liveness, startup

- Task: on Deployment `frontend` (project `myapp`), add a readiness probe on
  `/` and a liveness probe on `/` over HTTP, then watch the pod become ready.
- Expected outcome: `oc get deploy -o yaml` shows the probe blocks; pod
  `Ready` becomes `1/1`; `oc describe pod` shows the probes with no restart
  count climbing.

```sh
oc project myapp
oc set probe deploy/frontend --readiness --get-url=http://:80/ --period-seconds=5
oc set probe deploy/frontend --liveness --get-url=http://:80/ --period-seconds=10
```

- Verification:

```sh
oc get deploy frontend -o yaml | grep -A8 -E "readinessProbe|livenessProbe"
oc get pods -l app=frontend
oc describe pod -l app=frontend | grep -E "Liveness|Readiness|Restart Count"
```

### Exercise 5 — HorizontalPodAutoscaler

- Task: enable the autoscaler on `frontend` (1–5 replicas, 50% CPU), then check
  the HPA object and the metric it reads.
- Expected outcome: `oc get hpa frontend` shows target CPU 50%, min 1 / max 5,
  and `oc autoscale` reports success. (You are verifying the wiring, not
  generating load.)

```sh
oc autoscale deployment frontend --min=1 --max=5 --cpu-percent=50
```

- Verification:

```sh
oc get hpa frontend
oc describe hpa frontend
oc get deploy frontend -o wide
```

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `application-deployment` in the `ex288-developer` bank.
- **First pass**: **Training** mode — read every explanation, especially
  rollout-history and probe semantics.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: re-sit `application-deployment` Mastery after Module 3 — weighted
  route canaries will use these same workloads.

If `oc rollout` commands feel slow, drill Exercises 2–3 back to back until the
`rollout status/history/undo/latest` verbs come automatically.

## See also

- Red Hat EX288 exam page: redhat.com/training/ex288
- [Developer path index](README.md) — next: Module 2, Building applications —
  BuildConfigs, S2I, ImageStreams, triggers.
