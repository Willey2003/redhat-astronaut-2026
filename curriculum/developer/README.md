# EX288 — OpenShift Application Development: Learning Path

Target: **Red Hat Certified Specialist in OpenShift Application Development
(EX288)**. This path builds the skills of a developer deploying, building,
exposing, and automating applications on OpenShift: getting code into images,
getting images into workloads, getting workloads in front of users, and wiring
the whole thing to CI/CD.

## Prerequisites for this path

- The [global setup](../README.md) — `oc` CLI and a running cluster.
- Comfort with YAML, git, and basic container concepts (images, registries,
  Dockerfiles).
- A working knowledge of Deployments, Services, and pods. If you have not done
  so, skim the first two admin modules (`admin/01`, `admin/02`) — the developer
  path assumes you can read `oc get`/`oc describe` output and understand
  projects and ServiceAccounts.

## The four modules

| # | Module | Focus | Bank domains |
|---|---|---|---|
| [01](01-application-development.md) | Deployment models: Deployments, DeploymentConfigs, scaling, probes | Rollouts, self-healing, scaling, health checking | `application-deployment` |
| [02](02-application-development.md) | Building applications: BuildConfigs, S2I, ImageStreams, triggers | Source-to-image, builds, image tracking, webhooks | `application-build` |
| [03](03-application-development.md) | Routing and services for apps: Routes, ConfigMaps, Secrets | TLS routes, path/weighted routing, config injection | `application-routing` |
| [04](04-application-development.md) | GitOps and CI/CD: Pipelines, Argo CD, helm | Tekton, GitOps sync, chart rendering, `oc apply` | `gitops-cicd` |

## How to study

1. Work the modules in order — Module 3's weighted Routes build on Module 1's
   Deployments, and Module 4's pipelines consume the builds from Module 2.
2. Run every hands-on exercise against your cluster. The simulator grades its
   hands-on questions with the same kind of `oc` checks, so command fluency is
   exam currency.
3. After each module, sit a **Training** attempt focused on that module's bank
   domain (see each module's "Test yourself" block), then a **Mastery** attempt
   after every two modules.
4. Finish with a full-bank Training, then Mastery, then timed **Exam**-mode
   dress rehearsals. See the [schedule](../README.md) for timing.

## What the exam actually checks

EX288 is a hands-on exam: you are given a cluster and asked to deploy, build,
expose, and configure applications — rolling updates, probes, S2I builds,
routes in every TLS mode, config injection, and pipeline/GitOps-style delivery —
without reference material. Speed and recall matter; the `banks/ex288-developer`
bank mirrors this with a mixed engine, a 180-minute duration, and a 0.70 pass
threshold.

## See also

- Red Hat EX288 exam page: redhat.com/training/ex288
- [Golden Astronaut 2026 learning overview](../README.md)
