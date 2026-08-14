# EX280 — OpenShift Administration: Learning Path

Target: **Red Hat Certified Specialist in OpenShift Administration (EX280)**.
This path builds the cluster-side skills the exam rewards: knowing how
OpenShift 4 is architected and installed, how access and security are modeled,
and how networking and storage are provisioned — all verified against a real
cluster.

## Prerequisites for this path

- The [global setup](../README.md) — `oc` CLI and a running cluster
  (OpenShift Local/CRC or OKD).
- Comfort with YAML and the Linux command line.
- You do **not** need to be a developer; the app-build modules live in the
  developer path. That said, module 5 uses workloads that consume storage, so a
  passing familiarity with Deployments helps.

## The five modules

| # | Module | Focus | Bank domains |
|---|---|---|---|
| [01](01-cluster-administration.md) | Cluster architecture, install concepts, and the oc CLI | Control plane, cluster operators, install models, working the API | `cluster-configuration` |
| [02](02-cluster-administration.md) | Projects, RBAC, users and ServiceAccounts | Namespaces-as-projects, roles and bindings, identity, SA tokens | `authentication-security` |
| [03](03-cluster-administration.md) | SCC, pod security, quotas and limits | Security Context Constraints, Pod Security Admission, ResourceQuota/LimitRange | `authentication-security` |
| [04](04-cluster-administration.md) | Networking: Services, Routes, NetworkPolicies | Service types, external exposure, east-west isolation | `networking` |
| [05](05-cluster-administration.md) | Storage and Operators | PV/PVC/StorageClasses, dynamic provisioning, Operator Lifecycle Manager | `storage`, `operators` |

## How to study

1. Work the modules in order — 02 needs 01's CLI skills, 04 needs 02's RBAC
   concepts for network policy examples, 05 assumes you can create workloads.
2. Run **every** hands-on exercise. Each names concrete resources and a
   verification command; the simulator's hands-on questions are graded with the
   same kind of `oc` checks.
3. After each module, sit a **Training** attempt focused on that module's bank
   domain (see each module's "Test yourself" block), then a **Mastery** attempt
   after every two modules.
4. Finish the path with a full-bank Training attempt, then full Mastery, then
   timed Exam-mode dress rehearsals. See the [schedule](../README.md) for
   timing.

## What the exam actually checks

EX280-style tasks are predominantly hands-on: you are given a cluster and asked
to perform administrative work — grant access, secure workloads, expose and
restrict traffic, provision storage, install and operate Operators — usually
without reference material. Speed and command recall matter as much as
understanding. The bank `banks/ex280-admin` mirrors this with a mixed
knowledge/hands-on engine, a 180-minute duration, and a 0.70 pass threshold.

## See also

- Red Hat EX280 exam page: redhat.com/training/ex280
- [Golden Astronaut 2026 learning overview](../README.md)
