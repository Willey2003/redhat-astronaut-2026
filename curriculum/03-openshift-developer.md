# Module 3 — OpenShift application development (EX282 · EX480)

Module 3 is the developer half of the platform. EX282 is the core
application-development certification: taking source code, turning it into
images and deployments, and exposing it. EX480 is the specialist certification
for the application developer — pipelines, GitOps, services, secrets, and
observability from the application's point of view. You need the same cluster
as Module 2 (CRC or OKD) plus `tkn` and `helm` installed. You should already
be able to deploy and expose a workload; now you will learn how OpenShift
builds it.

## Learning objectives

1. Create, update, and inspect workloads: Deployments and DeploymentConfigs,
   scaling, rollouts, and rollbacks.
2. Build and deploy applications from source: BuildConfigs, Source-to-Image
   (S2I), ImageStreams, and build triggers.
3. Configure applications with ConfigMaps, Secrets, environment variables, and
   probes.
4. Expose services with Routes and manage traffic: blue/green, canary, and
   the Route/Split concept.
5. Automate delivery with OpenShift Pipelines (Tekton) and GitOps with Argo CD.
6. Debug and observe applications: logs, `oc exec`, `oc port-forward`,
   Prometheus metrics, and pod health.

## Key concepts

- **Deployment vs DeploymentConfig**: a Deployment is the Kubernetes-native
  object; a DeploymentConfig adds OpenShift concepts — triggers and automatic
  rollouts on ImageStream or ConfigChange. `oc new-app` creates a
  DeploymentConfig by default; `oc create deployment` creates a Deployment.
- **Rollouts**: `oc rollout status dc/web`, `oc rollout undo dc/web`,
  `oc rollout history dc/web`. DeploymentConfigs expose image-change and
  config-change triggers that start rollouts automatically.
- **BuildConfig and S2I**: a BuildConfig (`kind: BuildConfig`) drives a
  build — strategy `Source`, `Docker`, or `Pipeline` (deprecated). S2I
  (`Source`) takes a git repo + builder image and produces an application
  image. `oc new-app --image-stream=php --code=git@...` is the fast path.
- **ImageStreams**: a pointer to images, local or pulled from a registry,
  with tags. `oc get imagestreamtag`. A build outputs to an ImageStream tag;
  a deployment watches that tag and rolls when it changes.
- **Build triggers**: `webhook` (GitHub/GitLab), `imageChange`,
  `configChange`. `oc describe bc/web` shows the triggers; `oc set triggers`
  modifies them.
- **ConfigMaps and Secrets**: `oc create configmap app-conf --from-file=...`,
  `oc create secret generic db-pass --from-literal=password=...`. Mount as
  files or inject as env. Secrets are base64 in etcd — treat them as
  sensitive anyway.
- **Probes**: `readinessProbe`, `livenessProbe`, `startupProbe` on HTTP,
  TCP, or exec. `oc set probe deployment web --readiness --http-get=/healthz
  --port=8080`. A failing readiness probe removes the pod from Service
  endpoints; a failing liveness probe restarts it.
- **Routes for traffic**: edge/passthrough/reencrypt TLS, `oc create route
  edge web --service=web`. Traffic management: run two deployments and switch
  a Route's service, or use `spec.alternateBackends`/weights for canary splits.
- **OpenShift Pipelines (Tekton)**: `tkn pipeline list`, `tkn pipelinerun
  logs -f`. A `Pipeline` chains `Task`s; a `PipelineRun` executes it. `oc
  create -f pipeline.yaml` in `openshift-pipelines` namespace.
- **GitOps with Argo CD**: the application repo is the source of truth. An
  `Application` CR points at a git repo + path; Argo CD syncs it to the
  cluster. `argocd app sync myapp`, `argocd app get myapp`.
- **Helm**: `helm create`, `helm install myapp ./chart`, `helm upgrade`,
  `helm rollback`. Charts templatize Kubernetes manifests; `oc` serves as the
  release target.
- **Observability**: `oc logs -f pod/web-...`, `oc exec -it pod/web -- sh`,
  `oc port-forward svc/web 8080:80`, Prometheus metrics via
  `oc get --raw /api/v1/...` or the monitoring stack, `oc describe pod` for
  events.

## Hands-on exercises

Requires CRC or OKD (`oc login` as `kubeadmin`). Install `tkn` and `helm`
first.

### Exercise 1 — New app from source with S2I

- Task: create an application from a sample Node.js repo using a builder
  image, then watch the build and rollout complete.
- Expected outcome: a BuildConfig runs to completion, an ImageStream gets a
  fresh tag, and the DeploymentConfig rolls out the new image.

```sh
oc new-project dev3
oc new-app nodejs:16-ubi8~https://github.com/sclorg/nodejs-ex.git --name=nodeapp
oc get builds
oc logs -f build/nodeapp-1
oc rollout status dc/nodeapp
oc get pods -l deploymentconfig=nodeapp
```

- Verification (build complete, pod ready):

```sh
oc get bc,is,dc
oc get pods -l deploymentconfig=nodeapp
```

### Exercise 2 — ImageStream-triggered rollout

- Task: tag a new image into the ImageStream that the DeploymentConfig
  watches and confirm an automatic rollout.
- Expected outcome: tagging `nodeapp:latest` to a new version triggers a new
  DeploymentConfig rollout without manual intervention.

```sh
oc tag nodeapp:latest nodeapp:stable
# simulate a new image push to the watched tag:
oc tag openshift/nodejs:16 nodeapp:latest
oc rollout status dc/nodeapp
oc rollout history dc/nodeapp
```

- Verification (a second rollout revision appears):

```sh
oc rollout history dc/nodeapp
oc get pods -l deploymentconfig=nodeapp
```

### Exercise 3 — Config, secrets, and probes

- Task: attach a ConfigMap and Secret to the app, mount them, and add
  readiness and liveness probes.
- Expected outcome: the deployment restarts cleanly with the mounts; probes
  are visible in `oc describe`.

```sh
oc create configmap app-conf --from-literal=MODE=prod
oc create secret generic db-pass --from-literal=PASSWORD=s3cr3t
oc set env dc/nodeapp --from=configmap/app-conf --prefix=CFG_
oc set env dc/nodeapp --from=secret/db-pass --prefix=DB_
oc set probe dc/nodeapp --readiness --http-get=/ --port=8080
oc set probe dc/nodeapp --liveness --http-get=/ --port=8080
oc rollout status dc/nodeapp
oc describe dc/nodeapp | grep -A3 Readiness
```

- Verification (env vars present in the running pod):

```sh
oc exec $(oc get pods -l deploymentconfig=nodeapp -o name | head -1) -- env | grep -E 'CFG_|DB_'
```

### Exercise 4 — Route and canary traffic split

- Task: deploy a second version, point a Route at the primary, then add a
  weighted alternate backend so 20% of traffic hits the canary.
- Expected outcome: the Route serves both versions and the alternate backend
  weight is set.

```sh
oc new-app nginx:1.24 --name=green
oc new-app nginx:1.25 --name=blue
oc expose svc/green --name=canary-route --port=80
oc set route-backends canary-route green=80 blue=20
oc get route canary-route -o jsonpath='{.spec.alternateBackends}{"\n"}'
curl -skI http://$(oc get route canary-route -o jsonpath='{.spec.host}')
```

- Verification (weights 80/20 shown, route responds):

```sh
oc get route canary-route -o jsonpath='{.spec.alternateBackends[*].weight}{"\n"}'
```

### Exercise 5 — Pipeline and GitOps

- Task: define a Tekton Pipeline that clones a repo, builds, and pushes an
  image; run it with `tkn`. Then create an Argo CD Application from a git
  directory and sync it.
- Expected outcome: the PipelineRun completes with all steps `Succeeded`, and
  `argocd app get` shows the app `Synced` and `Healthy`.

```sh
oc new-project pipes
oc create -f https://raw.githubusercontent.com/<your-pipeline-yaml>/pipeline.yaml
tkn pipeline start build-deploy --param repo=<your-git-repo> -w name=ws,claimName=pvc -n pipes
tkn pipelinerun logs -f -n pipes -L
# GitOps:
oc new-project gitops
argocd app create myapp --repo <git-url> --path manifests --dest-server https://kubernetes.default.svc --dest-namespace gitops
argocd app sync myapp
argocd app get myapp
```

- Verification (last PipelineRun succeeded; app healthy):

```sh
tkn pipelinerun list -n pipes
argocd app get myapp | grep -E 'Sync|Health'
```

## Test yourself

When you can do Exercises 1–5 without the book, sit simulator attempts:

- **Bank**: `banks/ex282-developer`
- **First pass**: **Training** mode, `focus_domain = building-deploying-applications`.
- **Second pass**: **Mastery** mode on `developer-workflows` and
  `pipelines-gitops`, timed.
- **Third pass**: **Training** on `banks/ex480-developer` →
  `advanced-deployments` and `configuration-secrets`.
- Aim for 80%+ on the Mastery passes before starting Module 4; below that,
  redo Exercises 1–3, which carry the build/rollout mechanics.

## Self-check quiz

1. **Q**: How does an ImageStream trigger a new rollout? **A**: The
   DeploymentConfig watches an ImageStream tag; when a build or `oc tag`
   updates it, the imageChange trigger starts a rollout.
2. **Q**: What is the difference between readiness and liveness probes?
   **A**: Readiness decides whether the pod receives traffic (removed from
   endpoints when failing); liveness decides whether the pod is restarted.
3. **Q**: How do you split traffic between two versions with one Route?
   **A**: Set a primary Service plus `alternateBackends` with weights (for
   example `green=80, blue=20`) for a canary split.
4. **Q**: What does `oc new-app --code=<git-url>` do? **A**: It creates a
   BuildConfig (S2I from a builder image), an ImageStream, a DeploymentConfig,
   and a Service for the source repo in one shot.
5. **Q**: In GitOps, what reconciles the cluster to git? **A**: An Argo CD
   Application — it continuously compares live state to the manifests in the
   repo and syncs (or reports) any drift.

## See also

- EX282 and EX480 certification pages on redhat.com (referenced by name).
- docs.openshift.com — BuildConfigs, ImageStreams, S2I, Pipelines, and Argo CD
  chapters.
- `oc explain buildconfig`, `man tkn`, `helm help`.
- [Module 4 — Advanced administration, install, virtualization (EX380 · EX432 · EX316)](04-advanced-admin.md)
  — next: scale, upgrade, and virtualize what you just learned to build.
