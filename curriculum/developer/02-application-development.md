# Developer Module 2 — Building applications: BuildConfigs, S2I, ImageStreams, triggers

Module 1 got prebuilt images running. This module covers the OpenShift way of
*producing* images from source: BuildConfigs with Source-to-Image and Docker
strategies, ImageStreams for tracking what changed, and the triggers that make
rebuilds and redeploys automatic.

## Learning objectives

1. Create builds from git source with the Source-to-Image (S2I) and Docker
   strategies using BuildConfigs.
2. Run, follow, and inspect builds with the CLI.
3. Use ImageStreams and tags to track image versions and promote them between
   projects.
4. Trigger builds via config changes, image changes, webhooks, and `oc
   start-build`.

## Key concepts

- **BuildConfig**: the OpenShift resource describing a build — the source
   (git URL, binary upload, or Dockerfile), the strategy (`Source`/S2I,
   `Docker`, `Custom`, `Pipeline` — the last is deprecated), and the output
   (an ImageStreamTag). Each run materializes a `Build` object.
- **S2I (Source strategy)**: a builder image assembles your source into a new
   image — no Dockerfile needed. Pick a builder matching your runtime, e.g.
   `registry.access.redhat.com/ubi8/nodejs-18` with a Node.js repo, or
   `registry.access.redhat.com/ubi8/nginx-120` with static files.
- **Docker strategy**: the build runs your repository's `Dockerfile`. Useful
   for custom runtimes. `--strategy=docker` on `oc new-build`.
- **The build lifecycle**: `New → Pending → Running → Complete/Failed`.
   `oc logs -f bc/<name>` streams the build pod logs; `oc describe bc` shows
   triggers and webhook URLs; `oc get builds` lists runs.
- **Creating builds**: `oc new-build <builder>~<git-url>` (S2I shorthand),
   `oc new-build --context-dir=... --name=...`, `oc start-build <name>
   --from-dir=.` (binary build from a local directory).
- **ImageStream and tags**: an ImageStream tracks images by tag
   (`oc get is`, `oc tag <istag> <istag2>`, `oc import-image`). It decouples
   "what to watch" from "what image is deployed", enabling triggers and
   promotion without pulling digests.
- **ImageChangeTrigger**: redeploys a workload (and rebuilds an image) when an
   ImageStreamTag updates — the engine behind "commit → build → redeploy".
- **ConfigChangeTrigger**: rebuilds when the BuildConfig itself changes.
- **Webhook triggers**: GitHub/GitLab/Generic webhooks with a secret, exposed
   as a URL on the BuildConfig. A POST (e.g. a push event) starts a build:
   `curl -k -X POST <webhook-url>`.
- **Build run policy**: `Serial` (default, one at a time), `SerialLatestOnly`
   (queue only the newest), `Parallel`. Relevant when many commits land at once.
- **Integrated registry**: `image-registry.openshift-image-registry.svc:5000`,
   exposed via the `default-route`. `oc registry login` and
   `oc import-image <istag> --from=<external> --confirm` move images in and
   out.
- **Where builds run**: on a dedicated build pod in your namespace. The build
   needs a pull secret for private sources/registries and a push target it can
   reach; on OpenShift Local everything is local.

## Hands-on exercises

### Exercise 1 — S2I build from git

- Task: in project `builder`, create an S2I BuildConfig for a Node.js sample
  repo using the UBI Node.js builder image, then follow the build to
  completion.
- Expected outcome: `oc get builds` shows a `Complete` build and an ImageStream
  `nodejs-ex` with a `latest` tag.

```sh
oc new-project builder
oc new-build registry.access.redhat.com/ubi8/nodejs-18~https://github.com/sclorg/nodejs-ex.git --name=nodejs-ex
oc logs -f bc/nodejs-ex
```

- Verification:

```sh
oc get builds -o wide
oc get is nodejs-ex
oc get imagestreamtag nodejs-ex:latest
oc get bc nodejs-ex
```

### Exercise 2 — Deploy what you built

- Task: deploy the image your S2I build produced into project `builder` and
   verify the app responds.
- Expected outcome: a Deployment from ImageStreamTag `nodejs-ex:latest` runs,
   and a curl to the exposed Service returns the Node.js sample page.

```sh
oc new-app nodejs-ex:latest --name=nodejs-app
oc expose svc/nodejs-app
oc rollout status deploy/nodejs-app --watch
```

- Verification:

```sh
oc get route nodejs-app
curl -s http://$(oc get route nodejs-app -o jsonpath='{.spec.host}') -o /dev/null -w '%{http_code}\n'
```

### Exercise 3 — Binary build from a local directory

- Task: create a local `src/` folder with an `index.html`, then run a binary
   build of the nginx builder into ImageStream `static-site:latest`.
- Expected outcome: a build completes from local content and the resulting tag
   exists in the `static-site` ImageStream.

```sh
mkdir -p /tmp/binbuild && echo "<h1>hello from binary build</h1>" > /tmp/binbuild/index.html
oc new-build registry.access.redhat.com/ubi8/nginx-120 --name=static-site --binary
oc start-build static-site --from-dir=/tmp/binbuild --wait
```

- Verification:

```sh
oc get is static-site
oc get imagestreamtag static-site:latest -o jsonpath='{.image.dockerImageMetadata.Config.Labels.version}{"\n"}'
oc get builds | grep static-site
```

### Exercise 4 — ImageStream tag promotion and image change trigger

- Task: tag `static-site:latest` as `static-site:prod`, then create a
   DeploymentConfig that triggers on the `prod` tag and confirm a new
   deployment fires when the tag image changes.
- Expected outcome: `oc tag static-site:latest static-site:prod` succeeds; a DC
   with `imageChangeParams.from` on `prod` redeploys when the tag is replaced.

```sh
oc tag static-site:latest static-site:prod
oc apply -f - <<'EOF'
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: site
  namespace: builder
spec:
  replicas: 1
  triggers:
    - type: ConfigChange
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames: [site]
        from:
          kind: ImageStreamTag
          name: static-site:prod
  selector:
    app: site
  template:
    metadata:
      labels:
        app: site
    spec:
      containers:
        - name: site
          image: static-site:prod
          ports:
            - containerPort: 8080
EOF
```

- Verification:

```sh
oc rollout status dc/site --watch
oc get dc site -o jsonpath='{.spec.triggers[*].type}{"\n"}'
oc describe dc site | grep -A3 "Image Change"
```

### Exercise 5 — Webhook-triggered build

- Task: retrieve the generic webhook URL for `nodejs-ex`, POST a JSON payload
   to it, and confirm a new build starts.
- Expected outcome: `oc get builds` shows a fresh build in `Running`/`Pending`
   shortly after the POST.

```sh
URL=$(oc describe bc nodejs-ex | grep "Webhook Generic" | awk '{print $4}')
curl -k -X POST -H "Content-Type: application/json" -d '{}' "$URL"
```

- Verification:

```sh
oc get builds | head -5
oc get bc nodejs-ex -o jsonpath='{.spec.triggers[?(@.type=="Generic")].generic.secretReference.name}{"\n"}'
```

> Webhooks fire into the BuildConfig's trigger endpoint; if the cluster has no
> route into the API, use the internal service URL or test the trigger by
> calling `oc start-build nodejs-ex` instead — the graded question in the bank
> checks that a trigger exists and points at the right secret, not that your
> laptop can reach the cluster from outside.

## Test yourself

When Exercises 1–5 are smooth:

- **Domain**: `application-build` in the `ex288-developer` bank.
- **First pass**: **Training** mode — study explanations on S2I vs Docker
   strategy and trigger semantics.
- **Second pass**: **Mastery** mode, timed, focused on the domain.
- **Spiral**: re-sit `application-deployment` Mastery (Module 1); builds feed
   workloads, and the exam mixes both domains in one scenario.

If builds fail at pull time, check the builder image name and your registry
reachability; on OpenShift Local use `registry.access.redhat.com` images (no
pull secret needed) and re-run Exercise 1.

## See also

- Red Hat EX288 exam page: redhat.com/training/ex288
- [Developer path index](README.md) — next: Module 3, Routing and services for
   apps — Routes, ConfigMaps and Secrets.
