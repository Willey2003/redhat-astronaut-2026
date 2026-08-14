# Golden Astronaut 2026 — Learning Curriculum

This directory is the guided study path for the **EX280** (OpenShift
Administration) and **EX288** (OpenShift Application Development) certification
track. It is not a topic dump: it tells you **what** to study, **in what order**,
and **when** to sit which practice attempt against the Golden Astronaut 2026
simulator and your real cluster.

Two parallel paths live here:

| Path | Target | Modules | Bank |
|---|---|---|---|
| [admin/](admin/README.md) | EX280 — OpenShift Administration | 5 (admin/01..05) | `banks/ex280-admin` |
| [developer/](developer/README.md) | EX288 — OpenShift Application Development | 4 (developer/01..04) | `banks/ex288-developer` |

---

## How to use this curriculum

Work the two paths **in parallel**, one module per track per week (see the
8-week schedule below). For every module:

1. Read the **learning objectives** and the **key concepts**.
2. Do the **hands-on exercises** against your real cluster (CRC/OpenShift Local
   or OKD). Every exercise has a concrete task, an expected outcome, and the
   exact `oc` command to verify it.
3. Finish with the **test yourself** block: sit a simulator attempt on that
   module's bank domain, starting in **Training** mode and escalating to
   **Mastery**.

The modules are numbered within each path, so follow them top to bottom. Later
modules assume the skills of earlier ones.

## Prerequisite skills

You should be comfortable with the following **before** starting Week 1:

- **Linux command line** — navigating the filesystem, running commands, piping
  output, editing files (`vim` or similar).
- **YAML** — indentation, mappings, and lists; you will write and read a lot of
  it.
- **Container basics** — what an image is, images vs containers, registries,
  tags, and a rough idea of what a `Dockerfile` is.
- **Networking fundamentals** — DNS names, ports, TCP, HTTP, and TLS at a
  conceptual level.
- **git** — clone, commit, push; used for source-to-image builds and GitOps.

No prior OpenShift experience is required, but familiarity with Kubernetes
objects (pods, deployments, services) makes the early modules go faster.

## Tooling setup

Install and verify these before your first attempt. Run `./ga doctor` in the
repo root for a preflight check.

- **`oc` CLI** — the OpenShift client. Install the version that matches your
  cluster. On OpenShift Local the version is shown on the OpenShift web console
  **Help > Command Line Tools** page; you can also grab it from the OpenShift
  mirror. Verify with `oc version`.
- **A cluster** — one of:
  - **Red Hat OpenShift Local (CRC)** — the easiest local cluster. Minimum 4
    vCPU and 16 GiB RAM, more is better. Single node, includes the web console,
    integrated registry, and a built-in OperatorHub catalog. Default
    credentials are `kubeadmin` (the password is printed by `crc start`).
  - **OKD** — the community distribution, if you prefer an open-source build.
  - A sandbox / lab cluster if you have access to one. The exercises in this
    curriculum are written for a single-node local cluster and work on CRC.
- **Terminal** — a real terminal, not the web console's command line. You will
  run long `oc` commands and follow build/rollout logs.
- **Python 3 + Docker** — used by the Golden Astronaut platform itself
  (`./ga doctor`, `./ga up`).
- **Optional but useful**: `tkn` (Tekton/Pipelines CLI) and `helm` for the
  developer track's final module.

Once the cluster is up, log in:

```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
oc whoami
oc get nodes
```

## Modes and bank domains

The simulator serves attempts in three modes:

| Mode | Timer | Answer reveal | Best used for |
|---|---|---|---|
| **Training** | Off | Answers + explanations shown immediately | Learning each domain right after a module |
| **Mastery** | On | Hidden until graded | Practicing under time pressure without hints |
| **Exam** | On | Hidden until graded | Full dress rehearsal at real exam duration |

The banks are stratified by **domains** (declared in `banks/<bank>/exam.yaml`).
You can run an attempt restricted to one domain (`focus_domain`) — this is how
you target exactly what a module just taught. This is the recommended rhythm:

1. **After each module** → **Training** attempt, focused on that module's
   domain. Read every explanation, even for questions you got right.
2. **After every two modules** → **Mastery** attempt on those domains. No
   reveal; time yourself.
3. **After a full path** → one full-bank **Training** attempt, then one full
   **Mastery** attempt.
4. **The week before the exam** → repeated full-length **Exam** attempts until
   you score at or above the bank's `pass_threshold` (0.70) on three in a row.

If a domain shows up as a weakness on the post-attempt report, drill it with a
focused Mastery attempt and re-run the corresponding hands-on exercises.

## Recommended 8-week schedule (both tracks combined)

Roughly 10–12 hours per week. Weeks 1–4 introduce one module per track per
week; weeks 5–8 are consolidation, timed practice, and full rehearsal.

| Week | Admin (EX280) | Developer (EX288) | Practice attempts to sit |
|---|---|---|---|
| 1 | M1 — Architecture, install concepts, oc CLI | M1 — Deployments, DeploymentConfigs, scaling, probes | Training: `cluster-configuration`, `application-deployment` |
| 2 | M2 — Projects, RBAC, users, ServiceAccounts | M2 — BuildConfigs, S2I, ImageStreams, triggers | Training: `authentication-security`, `application-build`; Mastery: week-1 domains |
| 3 | M3 — SCC, pod security, quotas and limits | M3 — Routes, ConfigMaps and Secrets | Training: `authentication-security`, `application-routing`; Mastery: week-2 domains |
| 4 | M4 — Services, Routes, NetworkPolicies | M4 — Pipelines, Argo CD, helm | Training: `networking`, `gitops-cicd`; Mastery: week-3 domains |
| 5 | M5 — Storage, Operators and lifecycle | Review + full-path clean-up | Training: `storage`, `operators`, `application-lifecycle`; first full-bank **Training** on both banks |
| 6 | Weak-domain drill + re-run exercises | Weak-domain drill + re-run exercises | Full-bank **Mastery** on both banks; focused Mastery on your weakest domains |
| 7 | Time management practice | Time management practice | **Exam**-mode dry runs (any bank, focus on pacing and command speed) |
| 8 | Final rehearsal | Final rehearsal | Full-length **Exam** attempts until ≥ 0.70 on three consecutive attempts per track |

Adjust the pace: if week 3's security material is new to you, hold a full extra
week before moving on rather than rushing. The schedule is a floor, not a trap.

## Relationship to the simulator

- **Learning modules** teach the concepts and mechanics (`what` and `how`).
- **Hands-on exercises** build muscle memory against the real cluster (`do it
  yourself`).
- **Bank attempts** measure recall and readiness (`can you do it on demand`).
  Training mode is study material; Mastery is practice; Exam is evaluation.
  Treat a low Mastery score as a signal to redo exercises, not just to re-read
  the module.

The grader evaluates hands-on bank questions against your live cluster using
the same `oc` commands you see in the module exercises, so the exercises are
your best preparation for the hardest questions.

## See also

Official Red Hat certification exam pages (referenced by name — consult the
Red Hat site for current objectives, which change between releases):

- **EX280** — Red Hat Certified Specialist in OpenShift Administration exam:
  redhat.com/training/ex280
- **EX288** — Red Hat Certified Specialist in OpenShift Application Development
  exam: redhat.com/training/ex288
- Product documentation for the OpenShift 4 release you run is available from
  the OpenShift docs portal (docs.openshift.com) — use the "What's new" and
  exam-relevant sections as reference while studying.

## Contents

- [Admin path index](admin/README.md) — EX280 modules 01–05
- [Developer path index](developer/README.md) — EX288 modules 01–04
