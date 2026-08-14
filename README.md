# Red Hat Astronaut 2026

An independent, **from-scratch** exam simulator and learning platform for the
Red Hat / OpenShift **2026** certification track — built to be owned, extended,
and controlled entirely by you.

> Target certifications (2026 objectives):
> **RHCA** · **RHCE** · **EX280** · **EX316** · **EX380** · **EX480** ·
> **EX180** · **EX432** · **EX430** · **EX370** · **EX229** · **EX282** ·
> **RHCOA**
> (EX280 admin + EX282 developer paths, plus specialist tracks; RH024-style
> knowledge coverage.)

This project does **not** reuse the code of any other simulator. Everything —
engine, grading, bank format, UI, cluster automation — is original.

---

## Why it exists

The current kubestronaut-sim practice environment targets Kubernetes
certifications. Golden Astronaut 2026 covers the *OpenShift* track: Red Hat's
Kubernetes distribution, with its own administration and development
competencies — Operators, SCCs, Routes, BuildConfigs, ImageStreams, GitOps,
and more. It is a companion platform for the same learning journey.

## What the platform provides

| Layer | What you get |
|---|---|
| **Learning paths** | Objective-by-objective curricula for the admin (EX280) and developer (EX282) tracks, with per-module goals and practice exercises |
| **Exam simulator** | Full timed exam attempts, stratified question draws, Training / Mastery / Exam modes, behaviour-graded hands-on tasks |
| **Grading engine** | Original conductor that scores multiple-choice automatically and executes real checks (`oc`, `kubectl`, manifests) against your OpenShift cluster for hands-on tasks |
| **Cluster automation** | Scripts to provision a real OpenShift cluster — Red Hat OpenShift Local (CRC) or OKD — plus a preflight checker |
| **Knowledge base** | A multi-choice bank with full explanations for every question |

## Quick start

```bash
./ga doctor        # preflight: python, docker, oc, RAM, virtualization
./ga install       # build platform containers + write config
./ga up            # start the platform
./ga exam ex280-admin  # start a timed EX280 attempt (headless API) or open the UI
```

Open <http://localhost:8900> (or the LAN address after `ga expose`) in your
browser and pick a certification.

See [docs/install.md](docs/install.md), [docs/cli.md](docs/cli.md).

## Repository layout

```
banks/          Question banks in the project's own YAML format (banks/*/exam.yaml + qNN/*.yaml)
curriculum/     Learning-path content for the admin and developer tracks
engine/         Original simulator: facilitator (UI/API), conductor (grader), bank loader, CLI
labs/           Hands-on exercises and their grader checks
cluster/        OpenShift provisioning: CRC (OpenShift Local), OKD, preflight
docs/           Architecture, bank spec, install, security, roadmap
examples/       Reference banks, curriculum module, and grader checks
```

## Current status

**v1 (in progress):** platform skeleton, bank format + validator, engine,
curriculum, labs, cluster automation, verification. See [docs/roadmap.md](docs/roadmap.md).

## License

Apache-2.0. Original work, fully owned. Independent project — not affiliated
with Red Hat or the Linux Foundation. See [LICENSE](LICENSE).
