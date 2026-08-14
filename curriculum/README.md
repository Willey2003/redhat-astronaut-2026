# Red Hat Astronaut 2026 — Learning Curriculum

This directory is the guided study path for the **Red Hat Astronaut 2026**
platform. It is not a topic dump: it tells you **what** to study, **in what
order**, and **when** to sit which practice attempt against the simulator and
your real cluster. Work the modules top to bottom — later modules assume the
skills of earlier ones.

| Module | Title | Target certification | Bank | Engine |
|---|---|---|---|---|
| [01](01-containers-and-linux.md) | Containers and core Linux | EX180 | `banks/ex180` | knowledge |
| [02](02-openshift-admin.md) | OpenShift administration | EX280 · EX229 | `banks/ex280-admin`, `banks/ex229` | mixed · mixed |
| [03](03-openshift-developer.md) | OpenShift application development | EX282 · EX480 | `banks/ex282-developer`, `banks/ex480-developer` | mixed · mixed |
| [04](04-advanced-admin.md) | Advanced administration, install, virtualization | EX380 · EX432 · EX316 | `banks/ex380-admin`, `banks/ex432`, `banks/ex316` | mixed · knowledge · mixed |
| [05](05-automation-and-storage.md) | Automation and data storage | EX430 · EX370 | `banks/ex430-ansible`, `banks/ex370-storage` | mixed · mixed |
| [06](06-capstones.md) | Capstones: RHCE, RHCA, RHCOA | RHCE · RHCA · RHCOA | `banks/rhce`, `banks/rhca`, `banks/rhcoa` | knowledge |

## Why this order

The Red Hat track is a pyramid, and the curriculum climbs it. **Containers
first** (EX180): every later exam assumes you can pull, run, build, and debug
containers with `podman` and understand what Kubernetes does with them. Then
the two OpenShift seats of the platform — **admin** (EX280) and **developer**
(EX282) — cover the control plane, identity and security, networking, storage,
and the build/deploy loop. **EX380** extends admin with scaling, Operators,
and advanced networking, so it follows EX280 directly.

The specialist tracks hang off that core. **Virtualization** (EX316),
**install** (EX432), and **EX430** automation, **EX370** storage, and **EX229**
container management come after you own the platform — each is a "pick a lane"
deep dive. Finally the capstones certify the *whole* person: **RHCE** proves
you can automate RHEL with Ansible, **RHCA** proves you can design hybrid-cloud
architecture, and **RHCOA** proves you can run an OpenStack cloud. They come
last because they assume every other module's vocabulary.

## How to use this path

1. **Read the module.** Start with the learning objectives, then the key
   concepts. Concepts are ordered so that each one builds on the last.
2. **Do the exercises.** Every exercise has a concrete task, an expected
   outcome, and the exact command (`podman`, `oc`, `ansible`) to verify it.
   Do not skip verification — if the check fails, the cluster state is not what
   you think it is.
3. **Finish each module with its "test yourself" block.** Sit a simulator
   attempt on the module's bank, narrowed to that module's `focus_domain`.
   Start in **Training** mode (solutions shown), then redo it in **Mastery**
   mode (timed, no hints). Escalate to **Exam** mode only after several clean
   Mastery passes.
4. **Let the attempt report drive review.** It ranks your domains
   weakest-first. Re-run the matching exercises, then drill that domain with a
   focused Mastery attempt. Repeat until you score at or above the 0.70
   threshold on three consecutive full Exam attempts per bank.

## Prerequisite skills

Be comfortable with the following **before** starting Week 1:

- **Linux command line** — navigating the filesystem, redirecting and piping
  output, editing files with `vim`/`nano`.
- **YAML** — indentation, mappings, lists. Nearly every exercise writes or
  reads it, and it is the literal syntax of all three knowledge banks.
- **Container basics** — what an image is, images vs containers, registries
  and tags, and a rough idea of what a `Dockerfile` does.
- **git** — clone, commit, push. Needed for source-to-image builds, GitOps,
  and every Ansible role in the later modules.
- **Networking fundamentals** — IP addresses, ports, TCP, HTTP, DNS, and TLS
  at a conceptual level.

No Red Hat administration experience is required to start Module 1, but the
faster you can type a `podman` or `oc` command the further each module's
exercises take you.

## Tooling setup

Install these before your first attempt. Run `./ga doctor` in the repo root
for a preflight check.

- **`oc` CLI** — the OpenShift client, a superset of `kubectl`. On Red Hat
  OpenShift Local, download the matching version from the web console
  **Help > Command Line Tools** page.
- **`podman`** — the container engine used throughout Module 1 (Red Hat
  Container Toolkit): `dnf install podman`.
- **`ansible` + `ansible-core`** — needed from Module 5 onward for RHCE work.
- **`openstack` client** — the RHCOA CLI (`python-openstackclient`).
- **A cluster** — the OpenShift-heavy modules require one:
  - **Red Hat OpenShift Local (CRC)** — the easiest local cluster. Minimum
    4 vCPU and 16 GiB RAM. Single node, includes the web console.
  - **OKD** — the community distribution.
  - A sandbox/lab cluster if you have access.
  Module 1 works with **no cluster at all** (plain `podman`); Modules 2–4 and
  EX316 need CRC; the RHCOA module describes a minimal OpenStack environment
  and notes which exercises are theory-only.
- **Python 3 + Docker** — the platform stack itself (`./ga doctor`, `./ga up`).
- **Optional but useful**: `tkn` (Tekton), `helm`, `jq`, `virtctl`, `oc-mirror`.

Log in to the cluster once it is up:

```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
oc whoami
oc get nodes
```

## Simulator modes

Every bank can be attempted in three modes. They exist to be used **in that
order**:

| Mode | Timer | Answer reveal | Use it for |
|---|---|---|---|
| **Training** | Off | Solutions + explanations shown immediately | Learning each domain right after a module |
| **Mastery** | On | Hidden until graded | Practicing under time pressure without hints |
| **Exam** | On | Hidden until graded | Full dress rehearsal at real exam duration |

Attempts are drawn stratified by **domain**. You can restrict a Mastery or
Training attempt to a single domain with `focus_domain` — that is how you
target exactly what a module just taught. The domain names live in each bank's
`exam.yaml` and match the vocabulary below.

## Bank domains (the `focus_domain` vocabulary)

| Bank | Domains |
|---|---|
| `ex180` | `containers-podman`, `kubernetes-basics`, `building-images`, `container-security` |
| `ex280-admin` | `cluster-configuration`, `authentication-security`, `networking`, `storage`, `operators`, `application-lifecycle` |
| `ex229` | `cluster-administration`, `security-rbac`, `networking-cluster`, `storage-apps`, `logging-monitoring` |
| `ex282-developer` | `openshift-concepts`, `developer-workflows`, `building-deploying-applications`, `pipelines-gitops`, `debugging-troubleshooting` |
| `ex480-developer` | `application-development`, `advanced-deployments`, `services-networking`, `configuration-secrets`, `observability-apps` |
| `ex380-admin` | `advanced-cluster-configuration`, `scaling-performance`, `storage-advanced`, `networking-advanced`, `operators`, `security-compliance` |
| `ex432` | `cluster-installation`, `installation-methods`, `post-install-configuration`, `troubleshooting-install` |
| `ex316` | `virtualization-concepts`, `openshift-virtualization`, `vm-lifecycle`, `migration-networking` |
| `ex430-ansible` | `ansible-fundamentals`, `playbooks-templating`, `roles-collections`, `automation-controller`, `content-collections-execution` |
| `ex370-storage` | `storage-concepts`, `openshift-data-foundation`, `persistent-storage`, `backup-recovery` |
| `rhce` | `ansible-fundamentals`, `playbooks`, `roles`, `automation-controller`, `system-configuration` |
| `rhca` | `enterprise-architecture`, `hybrid-cloud-strategy`, `openshift-architecture`, `automation-at-scale`, `security-architecture` |
| `rhcoa` | `openstack-core`, `compute-nova`, `networking-neutron`, `storage-cinder`, `identity-keystone` |

> The `ex432` bank is expanded as its question set grows; the `draw_size`
> validator will fail until it has enough questions, which is expected while a
> bank is in progress. Everything else above validates clean.

## Reading your attempt report

After every Mastery or Exam attempt the score screen shows per-domain
performance. Treat domains below 70% as a backlog: do one focused Mastery
attempt per weak domain, and re-run the module exercises named in that
domain's row above. A weak domain is a command to **redo exercises**, not to
re-read the module — the grader checks the same real cluster (or command
line) the exercises used, so muscle memory is what scores.

## Recommended 12-week schedule

Roughly 10–12 hours per week. Weeks 1–8 introduce one module at a time; weeks
9–12 are consolidation and capstone preparation. Compress by merging the
specialist weeks you already know; stretch by giving virtualization or RHCA an
extra week — the schedule is a floor, not a trap.

| Week | Study | Hands-on | Simulator attempts (mode: bank → domain) |
|---|---|---|---|
| 1 | Module 1 — Podman, images, registries | Ex 1–2 (`podman run/pull/build`) | Training: `ex180` → `containers-podman`, `building-images` |
| 2 | Module 1 — Kubernetes basics + Linux admin | Ex 3–5 (`oc run`, systemd, users, dnf) | Training: `ex180` → `kubernetes-basics`, `container-security`; **Mastery**: `ex180` → `containers-podman` |
| 3 | Module 2 — EX280 architecture, projects, RBAC | Ex 1–2 (projects, roles, bindings) | Training: `ex280-admin` → `cluster-configuration`, `authentication-security` |
| 4 | Module 2 — networking, storage, Operators | Ex 3–5 (routes, NetworkPolicy, PVCs) | Training: `ex280-admin` → `networking`, `storage`; **Mastery**: `ex280-admin` → `authentication-security` |
| 5 | Module 3 — EX282 builds, S2I, pipelines | Ex 1–3 (BuildConfig, ImageStream, Pipeline) | Training: `ex282-developer` → `building-deploying-applications`, `pipelines-gitops`; **Mastery**: `ex280-admin` full-bank |
| 6 | Module 3 — EX480 GitOps, services, secrets | Ex 4–5 (Argo CD, Routes, ConfigMaps) | Training: `ex480-developer` → `advanced-deployments`, `configuration-secrets`; **Mastery**: `ex282-developer` → `developer-workflows` |
| 7 | Module 4 — EX380 scaling, Operators, compliance | Ex 1–3 | Training: `ex380-admin` → `scaling-performance`, `operators`; **Mastery**: `ex480-developer` full-bank |
| 8 | Module 4 — EX432 install + EX316 virtualization | Ex 4–5 (`openshift-install` concepts, `virtctl`) | Training: `ex432` → `cluster-installation`; Training: `ex316` → `openshift-virtualization`, `vm-lifecycle` |
| 9 | Module 5 — EX430 Ansible + AAP | Ex 1–3 (playbook, role, controller) | Training: `ex430-ansible` → `playbooks-templating`, `automation-controller`; **Mastery**: `ex380-admin` → `advanced-cluster-configuration` |
| 10 | Module 5 — EX370 OpenShift Data Foundation | Ex 4–5 (StorageClass, backup) | Training: `ex370-storage` → `openshift-data-foundation`, `backup-recovery`; **Mastery**: `ex430-ansible` full-bank |
| 11 | Module 6 — RHCE + RHCA strategy | Practice exams + drills | Training then **Mastery**: `rhce` → `playbooks`, `system-configuration`; Training: `rhca` → `enterprise-architecture` |
| 12 | Module 6 — RHCOA + full rehearsal | Re-run any failing exercise | Full-bank **Mastery** on weak banks; full-length **Exam** dry runs on `rhce`, `rhca`, `rhcoa`, and any open exam until ≥ 0.70 three times in a row |

## Reading your attempt report

After every Mastery or Exam attempt the score screen shows per-domain
performance. Treat domains below 70% as a backlog: do one focused Mastery
attempt per weak domain, and re-run the module exercises named in that
domain's row above.

## Relationship to the official certifications

The banks in `banks/` are original practice questions written to the published
exam objectives of the official Red Hat certifications, which are referenced
here **by name only** — always consult the Red Hat site for the current
objectives, since they change between exam releases: **EX180** (Containers and
Kubernetes), **EX280** (OpenShift Administration), **EX282** (OpenShift
Application Development), **EX380** (OpenShift Enterprise Administration),
**EX316** (OpenShift Virtualization), **EX432** (OpenShift Installation Lab),
**EX430** (Automation with Ansible Automation Platform), **EX370** (OpenShift
Data Administration), **EX229** (Managing Containers), **EX480** (OpenShift
Application Developer), **RHCE** (Red Hat Certified Engineer), **RHCA** (Red
Hat Certified Architect), and **RHCOA** (Red Hat Certified OpenStack
Administrator). Product documentation for the OpenShift release you run is
available from docs.openshift.com.

Red Hat Astronaut 2026 is an independent simulator, not affiliated with Red
Hat. Certification names are trademarks of their owners.

## Contents

- [Module 1 — Containers and core Linux (EX180)](01-containers-and-linux.md)
- [Module 2 — OpenShift administration (EX280 · EX229)](02-openshift-admin.md)
- [Module 3 — OpenShift application development (EX282 · EX480)](03-openshift-developer.md)
- [Module 4 — Advanced administration, install, virtualization (EX380 · EX432 · EX316)](04-advanced-admin.md)
- [Module 5 — Automation and data storage (EX430 · EX370)](05-automation-and-storage.md)
- [Module 6 — Capstones: RHCE, RHCA, RHCOA](06-capstones.md)
