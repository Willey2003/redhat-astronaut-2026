# Module 1 — Containers and core Linux (EX180)

Module 1 is the foundation of the whole track. EX180 is a knowledge-and-
practical certification about containers with **Podman** and Kubernetes
basics, sitting on top of core RHEL administration. Everything later — every
`oc` command in Modules 2–6 — assumes you can reason about images, runtimes,
and the workloads they hold. Nothing here needs a cluster: a RHEL-like host
with `podman` installed is enough.

## Learning objectives

1. Pull, inspect, run, and remove containers with `podman`, and explain how
   they differ from virtual machines.
2. Manage images: tags, digests, registries, `podman search/pull/rmi`, and
   storage.
3. Build images with a `Containerfile`/`Dockerfile` and publish them to a
   registry.
4. Run containerized workloads on Kubernetes/OpenShift: pods, deployments,
   services, and namespaces with `oc`.
5. Secure containers: rootless operation, SELinux, capabilities, and image
   provenance.
6. Perform core Linux administration on RHEL: `systemd`, user/group and file
   permission management, package management with `dnf`, and the `vim` editor.

## Key concepts

- **Container vs VM**: a container shares the host kernel, isolating
  processes with namespaces and resource limits with cgroups; a VM runs a full
  guest kernel. Containers start in milliseconds; VMs in seconds.
- **Podman**: a daemonless container engine. `podman run -d --name web
  -p 8080:80 nginx` runs nginx detached and publishes host port 8080 →
  container port 80. It is rootless by default — no daemon, just your user's
  namespaces.
- **Images, tags, digests**: `nginx:stable-alpine` is name:tag. Tags are
  mutable; digests (`nginx@sha256:...`) are content-addressed and immutable.
  `podman pull`, `podman push`, `podman tag`, `podman images` are the core
  image verbs.
- **Registries**: servers that store images — registry.access.redhat.com,
  quay.io, Docker Hub, or a private mirror. `podman search` queries them, and
  `/etc/containers/registries.conf` controls search and mirror order.
- **Containerfile**: the recipe for an image. `FROM ubi9/ubi`, `RUN`, `COPY`,
  `EXPOSE`, `USER`, `CMD`/`ENTRYPOINT`. Build with `podman build -t myapp:1.0
  .`; each instruction becomes a layer.
- **SELinux and rootless**: Podman sets SELinux labels on volumes
  (`:Z` relabels, `:z` shares) and runs unprivileged by default. If a mounted
  file looks empty, it is almost always an SELinux label problem:
  `ls -Z`, then `-v dir:dir:Z`.
- **Pods**: the smallest scheduling unit on Kubernetes/OpenShift — one or more
  containers sharing a network namespace and storage. `oc run web
  --image=nginx` creates a pod; `oc get pods -o wide` shows node and IP.
- **Deployments and Services**: a Deployment owns a ReplicaSet and declares
  desired replicas; a Service gives a stable ClusterIP in front of a
  label-selected set of pods. `oc create deployment web --image=nginx
  --replicas=3`, `oc expose deployment web --port=80`.
- **Namespaces (projects)**: the isolation boundary. `oc new-project app`,
  `oc get all -n app`. Quotas, RBAC, and NetworkPolicy all bind to a
  namespace.
- **systemd**: RHEL services run under systemd. `systemctl start|enable
  --now httpd`, `systemctl status`, `journalctl -u httpd -f` for logs.
- **Users and permissions**: `useradd`, `groupadd`, `usermod -aG`, and the
  octal/permission model. `visudo` manages sudoers.
- **Package management**: `dnf install|remove|update`, `dnf group install`,
  `dnf repolist`. DNF resolves dependencies and is idempotent at the command
  level (later, Ansible makes it idempotent at the *state* level).
- **vim**: the editor used in the labs. Practice `i`, `Esc`, `:wq`, `:q!`,
  `dd`, `yy`, `p` until they are muscle memory — you will live in it.

## Hands-on exercises

All Module 1 exercises run on a RHEL/Fedora host with Podman installed
(`dnf install podman`). **No OpenShift cluster or CRC is required.**

### Exercise 1 — Pull, run, and inspect containers

- Task: pull the `ubi9/ubi` image, run a hello container that prints the OS
  release, then inspect it.
- Expected outcome: `podman run --rm ubi9/ubi cat /etc/redhat-release` prints
  "Red Hat Enterprise Linux release 9.x"; `podman ps -a` lists it after a
  foreground run.

```sh
podman pull ubi9/ubi
podman run --rm ubi9/ubi cat /etc/redhat-release
podman images
podman inspect ubi9/ubi --format '{{.Os}} {{.Architecture}}'
```

- Verification (runs and exits cleanly, image present):

```sh
podman run --rm ubi9/ubi echo "container ok"
podman images | grep ubi9
```

### Exercise 2 — Publish ports and manage lifecycle

- Task: run nginx detached with a published port, verify it serves HTTP, then
  stop and remove it.
- Expected outcome: `curl http://localhost:8080` returns nginx HTML, and
  `podman ps` shows the container until you stop it.

```sh
podman run -d --name web -p 8080:80 nginx
curl -s http://localhost:8080 | head -1
podman ps
podman stop web && podman rm web
```

- Verification (port responds, container gone):

```sh
curl -sI http://localhost:8080 | head -1
podman ps -a --filter name=web
```

### Exercise 3 — Build an image from a Containerfile

- Task: write a Containerfile that copies a tiny HTML file into nginx and
  builds image `myweb:1.0`, then run it.
- Expected outcome: `podman build -t myweb:1.0 .` completes, and the running
  container serves your custom page.

```sh
mkdir -p ~/myweb && cd ~/myweb
printf '<h1>Module 1</h1>\n' > index.html
cat > Containerfile <<'EOF'
FROM nginx
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
podman build -t myweb:1.0 .
podman run -d --name web -p 8080:80 myweb:1.0
curl -s http://localhost:8080
```

- Verification (your page served; image listed):

```sh
curl -s http://localhost:8080 | grep "Module 1"
podman images | grep myweb
```

### Exercise 4 — Run a workload on Kubernetes

- Task: this one needs a cluster (CRC or OKD). Create a project, deploy nginx,
  scale it, and expose it with a Service and a Route.
- Expected outcome: `oc get pods` shows three ready pods, the Service shows a
  ClusterIP, and the Route is reachable.

```sh
oc new-project mod1
oc create deployment web --image=nginx:1.24 --replicas=3
oc scale deployment web --replicas=3
oc expose deployment web --port=80
oc get pods,svc,route -n mod1
```

- Verification (3/3 ready; route URL returns 200):

```sh
oc get pods -n mod1
curl -sI http://$(oc get route web -n mod1 -o jsonpath='{.spec.host}') | head -1
```

### Exercise 5 — Core Linux administration

- Task: create a service account user, a group, a config file owned by that
  group, install a package, and enable a service.
- Expected outcome: the user `svcapp` exists in group `svcgrp`; `/etc/app.conf`
  is group-readable by `svcgrp`; `httpd` is installed and `enabled`.

```sh
sudo groupadd svcgrp
sudo useradd -g svcgrp -m svcapp
echo "mode=prod" | sudo tee /etc/app.conf
sudo chgrp svcgrp /etc/app.conf && sudo chmod 640 /etc/app.conf
sudo dnf install -y httpd
sudo systemctl enable --now httpd
sudo systemctl status httpd --no-pager
```

- Verification (id, permissions, service state):

```sh
id svcapp
ls -l /etc/app.conf
systemctl is-enabled httpd && systemctl is-active httpd
```

## Test yourself

When you can do Exercises 1–5 without the book, sit a simulator attempt:

- **Bank**: `banks/ex180`
- **First pass**: **Training** mode, `focus_domain = containers-podman` —
  read the explanation for every question, right or wrong.
- **Second pass**: **Mastery** mode, same domain, timed.
- **Third pass (optional)**: **Training**, `focus_domain = building-images` and
  `container-security`, to close the image and security gaps before Module 2.
- Aim for 80%+ on the Mastery pass before starting Module 2; below that, redo
  Exercises 1–3, which carry most of the mechanics.

## Self-check quiz

1. **Q**: Why does a file mounted into a Podman container appear empty?
   **A**: Almost always SELinux labeling — the container cannot read the
   unlabeled file; add `:Z` (relabel) or `:z` (shared) to the volume.
2. **Q**: What is the difference between a tag and a digest?
   **A**: A tag is a mutable human label (`nginx:latest`); a digest is the
   immutable content hash of an image layer set (`nginx@sha256:...`).
3. **Q**: How does a Deployment update a set of pods?
   **A**: It drives a ReplicaSet — scaling or changing the pod template
   creates a new ReplicaSet and rolls pods over while `maxUnavailable` and
   `maxSurge` bound the change.
4. **Q**: Why is Podman called "daemonless"?
   **A**: It has no always-on daemon like the old Docker engine; each `podman`
   command talks directly to the container runtime in your user's namespaces,
   which makes it rootless-safe and easy to run in CI.
5. **Q**: Which `oc` object gives a pod a stable virtual IP?
   **A**: A Service of type `ClusterIP`, which selects pods by label; pods are
   then reachable as `service.project.svc.cluster.local`.

## See also

- EX180 certification page on redhat.com (referenced by name; objectives
  change between releases).
- `man podman-run`, `man podman-build` on your RHEL host.
- Red Hat Container Tools documentation (containers.github.io) for rootless
  and SELinux detail.
- [Module 2 — OpenShift administration (EX280 · EX229)](02-openshift-admin.md)
  — next: the control plane, RBAC, and the cluster you now know how to run
  workloads on.
