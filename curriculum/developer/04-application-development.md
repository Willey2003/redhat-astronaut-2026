# Developer Module 4 — GitOps and CI/CD: OpenShift Pipelines, Argo CD concepts, helm template + oc apply

The modern EX288-style developer delivers applications declaratively and
automatically. This module covers the three delivery pillars you will be asked
about: **OpenShift Pipelines (Tekton)** for CI, **GitOps / Argo CD (OpenShift
GitOps)** for continuous delivery, and **Helm** for packaging and rendering
manifests that you then hand to `oc apply`.

## Learning objectives

1. Create Tasks, Pipelines, and PipelineRuns with OpenShift Pipelines (Tekton)
   and follow their logs.
2. Explain GitOps principles and model an Argo CD Application that syncs a git
   repo into a cluster.
3. Render manifests from a Helm chart with `helm template` and apply them with
   `oc apply` — the no-server, exam-safe Helm flow.
4. Chain the pieces: pipeline builds and renders, GitOps reconciles, and you
   can trace a manifest from chart to running workload.

## Key concepts

- **OpenShift Pipelines = Red Hat's build of Tekton**. Core CRDs: `Task`
   (steps), `Pipeline` (ordered/parallel Tasks), `PipelineRun` (an execution),
   `TaskRun`, `ClusterTask` (cluster-wide Tasks), `Workspace` (shared
   storage/volumes between steps). `tkn` is the CLI; the operator is "OpenShift
   Pipelines" from OperatorHub.
- **Steps share workspaces, not filesystems**: each step is its own container;
   a `workspaces` entry (e.g. a PVC or `emptyDir`) is what carries artifacts
   between steps.
- **PipelineRun flow**: `oc apply -f task.yaml pipeline.yaml`, then `oc create
   -f pipelinerun.yaml`, watch with `oc get pipelinerun` / `tkn pipelinerun
   logs -f`. Failures surface as step conditions you read with `oc describe
   taskrun`.
- **GitOps**: the git repository is the single source of truth for desired
   state; an agent continuously reconciles the cluster to it, correcting
   drift. Declarative, auditable, automated.
- **Argo CD / OpenShift GitOps**: installed via the "Red Hat OpenShift GitOps"
   operator. Core objects: `Application` (source repo + destination cluster +
   sync policy), `Project` (scoping), `ApplicationSet`. Sync policies:
   manual or automatic; automated sync watches the repo and applies diffs;
   health checks report app status.
- **Helm without Tiller**: OpenShift has no Helm server-side component. The
   exam-safe flow is **render + apply**: `helm template <release> <chart> -f
   values.yaml --namespace <ns> | oc apply -f -`. No `helm install`, no RBAC
   surprises — pure declarative manifests.
- **Charts**: `helm repo add`, `helm repo update`, `helm show values`, charts
   produce templated manifests from `values.yaml`; `--set key=value` overrides
   on the command line.
- **Kustomize** (bonus): `oc apply -k <dir>` applies a kustomization — same
   declarative spirit, different tooling; good to recognize on the exam.
- **Putting it together**: CI = Tekton Pipeline (checkout, build with
   buildah/podman, push); CD = GitOps Application (repo to cluster, auto-sync);
   packaging = Helm (render once, apply everywhere). The platform's own
   operators install via the same declarative flow you saw in admin Module 5.
- **Verification everywhere**: `oc get pipelinerun`, `oc get applications`,
   `oc get deploy` after a helm apply.

## Hands-on exercises

### Exercise 1 — A minimal Tekton Task and PipelineRun

- Task: in project `pipeline`, create a simple Task that echoes a message,
   then run it and confirm it succeeds.
- Expected outcome: `oc get taskrun` shows `Succeeded: True` and the echo is
   visible in the logs.

```sh
oc new-project pipeline
oc apply -f - <<'EOF'
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: echo
spec:
  steps:
    - name: say
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: |
        echo "hello from tekton"
        exit 0
EOF
oc create -f - <<'EOF'
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  generateName: echo-run-
spec:
  taskRef:
    name: echo
EOF
```

- Verification:

```sh
oc get taskrun -o custom-columns=NAME:.metadata.name,SUCCEEDED:.status.conditions[0].status
tkn taskrun logs -f --last
```

### Exercise 2 — A Pipeline with two ordered Tasks sharing a workspace

- Task: define Tasks `write-msg` and `read-msg` and a Pipeline that runs them
   in order sharing a `messages` workspace, then run it.
- Expected outcome: `oc get pipelinerun` succeeds; the second task reads the
   file the first wrote, proving workspace sharing.

```sh
oc apply -f - <<'EOF'
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: write-msg
spec:
  workspaces:
    - name: data
  steps:
    - name: write
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: echo "shared-data" | tee /workspace/data/msg.txt
---
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: read-msg
spec:
  workspaces:
    - name: data
  steps:
    - name: read
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: cat /workspace/data/msg.txt
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: msg-flow
spec:
  workspaces:
    - name: shared
  tasks:
    - name: write
      taskRef: { name: write-msg }
      workspaces:
        - name: data
          workspace: shared
    - name: read
      taskRef: { name: read-msg }
      runAfter: [write]
      workspaces:
        - name: data
          workspace: shared
EOF
oc create -f - <<'EOF'
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: msg-flow-run-
spec:
  pipelineRef:
    name: msg-flow
  workspaces:
    - name: shared
      emptyDir: {}
EOF
```

- Verification:

```sh
oc get pipelinerun
tkn pipelinerun logs -f --last
oc get taskrun -o custom-columns=NAME:.metadata.name,SUCCEEDED:.status.conditions[0].status
```

### Exercise 3 — Helm template + oc apply

- Task: add a chart repo, render a chart with a custom release name and
   namespace into YAML, and apply the rendered output into project `helmapp`.
- Expected outcome: `helm template` prints manifests; `oc apply -f -` creates
   them; `oc get deploy` shows the chart's workload.

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
oc new-project helmapp
helm template myweb bitnami/nginx --namespace helmapp --set service.type=ClusterIP | oc apply -f -
```

- Verification:

```sh
helm template myweb bitnami/nginx --namespace helmapp --set service.type=ClusterIP | oc apply --dry-run=client -f - | tail -1
oc get deploy,svc -n helmapp
oc get deploy -n helmapp -o jsonpath='{.items[0].status.availableReplicas}{"\n"}'
```

### Exercise 4 — Render with values overrides

- Task: render the same chart with `--set replicaCount=3` and confirm the
   rendered Deployment requests three replicas before you apply it.
- Expected outcome: the rendered output contains `replicas: 3`; the applied
   Deployment runs 3 pods.

```sh
helm template myweb bitnami/nginx --namespace helmapp --set replicaCount=3 --set service.type=ClusterIP > /tmp/rendered.yaml
grep -A1 "^spec:" /tmp/rendered.yaml | head -5
oc apply -f /tmp/rendered.yaml
```

- Verification:

```sh
grep -c "replicas: 3" /tmp/rendered.yaml
oc get deploy myweb-nginx -o wide
oc get pods -n helmapp -l app.kubernetes.io/instance=myweb | wc -l
```

### Exercise 5 — Model an Argo CD Application

- Task: model the GitOps contract by applying an `Application` manifest that
   sources a repo and syncs into a namespace. (Offline, the sync will not
   complete without the GitOps operator and a reachable repo — the graded
   material is the *shape* of the Application and its sync policy.)
- Expected outcome: `oc apply` accepts the Application manifest and `oc get
   applications` lists it with the requested source, destination, and sync
   policy.

```sh
oc apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: helmapp
  project: default
  source:
    repoURL: https://github.com/example/myapp-deploy.git
    targetRevision: main
    path: manifests
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

- Verification:

```sh
oc get applications -n openshift-gitops
oc get application myapp -n openshift-gitops -o jsonpath='{.spec.source.repoURL}{" "}{.spec.destination.namespace}{"\n"}'
oc explain application.spec.syncPolicy.automated --recursive | head -10
```

> On a cluster where the OpenShift GitOps operator is not installed the CRD is
> missing and the apply fails — that is expected. Install the operator first
> (see admin Module 5 for the OLM flow) or use `oc explain` as the check that
> the API is present. The concept you are graded on is the Application
> manifest: repoURL, destination, project, and `syncPolicy.automated`.

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `gitops-cicd` in the `ex288-developer` bank.
- **First pass**: **Training** mode — study explanations on Pipeline
   workspaces and Argo CD sync policies.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: re-sit `application-build` Mastery (Module 2); a Tekton build
   step and an S2I BuildConfig are two answers to the same CI question, and the
   exam expects you to pick correctly per scenario.

This completes the developer path. Move to the full-bank Training and Mastery
attempts, then timed Exam-mode dress rehearsals per the global schedule.

## See also

- Red Hat EX288 exam page: redhat.com/training/ex288
- [Developer path index](README.md)
- [Admin Module 5](../admin/05-cluster-administration.md) — the OLM flow used
   to install the Pipelines and GitOps operators.
