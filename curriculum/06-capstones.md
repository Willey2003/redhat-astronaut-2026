# Module 6 — Capstones: RHCE, RHCA, RHCOA

Module 6 is the summit. The three knowledge banks in `banks/rhce`, `banks/rhca`,
and `banks/rhcoa` are the platform's capstones: **RHCE** (Red Hat Certified
Engineer) proves you can automate RHEL with Ansible end to end; **RHCA** (Red
Hat Certified Architect) proves you can design and govern open hybrid cloud
architecture; **RHCOA** (Red Hat Certified OpenStack Administrator) proves you
can operate an OpenStack cloud. Nothing in this module is new vocabulary —
every concept here appeared in Modules 1–5. What changes is the *level*:
knowledge questions ask you to choose between plausible answers, so the drills
target discrimination, not recall.

## Learning objectives

1. Explain RHCE automation design: idempotent playbooks, module choice,
   inventory design, and controller-based execution.
2. Justify architecture decisions at the RHCA level: capacity, availability,
   hybrid-cloud placement, and security layers.
3. Operate OpenStack services at the RHCOA level: projects, instances,
   networks, volumes, and identity.
4. Map every curriculum concept to its capstone domain and drill the weakest
   domain first.
5. Prepare with the Training → Mastery → Exam rhythm and a written exam-day
   strategy.

## Key concepts

- **RHCE = automation, not guessing**: every RHCE question rewards knowing
  *which module and which state* produce a given end state idempotently.
  Before choosing, ask: does this module check state? Will a second run be a
  no-op? `ansible.builtin.service`, `ansible.builtin.dnf`,
  `ansible.builtin.template`, `ansible.posix.firewalld`, and the
  `when`/`loop`/`block`/`rescue` control structures carry the exam.
- **RHCE inventory and configuration**: inventory is the host list; `-i` or
  the `inventory` setting in `ansible.cfg` points at it. Configuration
  precedence runs `ANSIBLE_CONFIG` > `./ansible.cfg` > `~/.ansible.cfg` >
  `/etc/ansible/ansible.cfg` — a favorite distractor.
- **RHCE roles**: the role directory layout (`tasks`, `handlers`, `defaults`,
  `vars`, `templates`, `files`) and variable precedence. Role defaults are the
  lowest precedence of all; play-level vars override them.
- **RHCE controller**: Projects sync from SCM, Credentials are stored
  encrypted and referenced by name, Job Templates bind playbook + inventory +
  credentials, and Execution Environments make runs reproducible.
- **RHCA = decisions, not commands**: RHCA questions present scenarios —
  "choose the architecture that meets RPO/RTO and cost" — and reward
  understanding *why* an option is right. Anchor every answer in a principle:
  availability, consistency, least privilege, portability.
- **RHCA availability**: RPO drives replication and backup frequency; RTO
  drives failover automation and standby capacity. Sizing tracks etcd/API load
  and cluster scale, with odd control plane counts for quorum.
- **RHCA hybrid cloud**: a consistent platform — the same OpenShift APIs and
  tooling — across on-premise, edge, and public clouds; ACM for fleet
  management; Single Node OpenShift for constrained edge sites.
- **RHCA security**: defense in depth — platform hardening, SCCs and Pod
  Security Admission, RBAC, NetworkPolicy, and a signed image supply chain,
  plus OAuth/OIDC identity integration and Compliance Operator evidence.
- **RHCOA = service mental model**: know which service owns what — Nova
  instances, Neutron networks, Cinder volumes, Glance images, Keystone
  identity and catalog, Horizon dashboard — and that every client speaks the
  same REST API through Keystone.
- **RHCOA day-2 operations**: projects as the isolation boundary, floating IPs
  for external access, routers for source NAT, security groups as port
  firewalls, snapshots as the pre-change safety net, and application
  credentials for scoped automation identity.
- **Drill method**: for each capstone bank, run a **Training** attempt focused
  on one domain and read every explanation; then a timed **Mastery** attempt;
  then a full-bank **Exam** attempt. Three consecutive Exam attempts at or
  above the 0.70 threshold is the pass bar you train to.

## Hands-on exercises

Exercise 1 needs a RHEL host and `ansible`; Exercises 2–3 are design and
drill work against the simulator; Exercise 4 needs an `oc`-reachable cluster;
Exercise 5 is an OpenStack walkthrough (theory-only where no cloud is
available).

### Exercise 1 — RHCE drill: idempotent system configuration

- Task: write a playbook that ensures a user, a package, a config file, and a
  running service — and prove the second run changes nothing.
- Expected outcome: `ansible-playbook` run twice; the first reports changes,
  the second reports `ok` for every task.

```sh
mkdir -p ~/cap && cd ~/cap
cat > hosts <<'EOF'
[all]
localhost ansible_connection=local
EOF
cat > site.yml <<'EOF'
- name: Idempotent baseline
  hosts: all
  become: yes
  vars:
    app_port: 9090
  tasks:
    - name: ensure user
      ansible.builtin.user:
        name: svc
        state: present
    - name: ensure package
      ansible.builtin.dnf:
        name: httpd
        state: present
    - name: render config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/httpd/conf.d/app.conf
      notify: restart httpd
    - name: ensure service
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: yes
  handlers:
    - name: restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
EOF
echo 'Listen {{ app_port }}' > app.conf.j2
ansible-playbook -i hosts site.yml
ansible-playbook -i hosts site.yml
```

- Verification (second run reports no `changed`):

```sh
ansible-playbook -i hosts site.yml | grep -c changed
```

### Exercise 2 — RHCA drill: scenario discrimination

- Task: for each scenario below, pick the principle that decides the answer,
  then confirm it against the bank's explanation.
- Expected outcome: you can state the principle in one sentence before looking
  at the options.

```sh
# Scenarios to run against banks/rhca in Training mode:
#  1. Choose a two-site deployment meeting RPO <= 1 min and RTO <= 15 min.
#  2. Pick the fleet-management tool for 100 clusters in 4 clouds.
#  3. Order the security layers for a PCI-scoped container platform.
#  4. Choose where an edge factory workload should run and why.
#  Use focus_domain = enterprise-architecture, hybrid-cloud-strategy,
#  security-architecture to isolate each.
```

- Verification (you name the principle first, then the option):

```sh
echo "RPO/RTO -> replication + failover automation"
echo "fleet -> ACM + GitOps placement"
echo "security -> defense in depth, least privilege"
echo "edge -> SNO, consistent platform"
```

### Exercise 3 — RHCOA drill: service map

- Task: map each task to its service by writing the service name, then check
  with the bank.
- Expected outcome: every mapping below is correct without hesitation.

```sh
echo "boot an instance         -> Nova (+ Glance image, flavor)"
echo "give it internet         -> Neutron (floating IP + router NAT)"
echo "persist its data         -> Cinder (volume, snapshot)"
echo "prove who may act        -> Keystone (roles, projects, tokens)"
echo "see it in a browser      -> Horizon (dashboard)"
```

- Verification (cross-check with `banks/rhcoa`, one domain per line):

```sh
echo "openstack-core, compute-nova, networking-neutron, storage-cinder, identity-keystone"
```

### Exercise 4 — Connect the tracks: automate OpenShift

- Task: write a playbook that deploys a namespace, a PVC, and a Deployment on
  your cluster using the `kubernetes.core.k8s` collection — the RHCE skill
  applied to the platform from Module 2.
- Expected outcome: the resources exist after the playbook runs.

```sh
pip install openshift   # or: ansible-galaxy collection install kubernetes.core
cat > deploy.yml <<'EOF'
- name: Deploy app to OpenShift
  hosts: localhost
  tasks:
    - name: ensure namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: cap-app
    - name: create PVC
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: data
            namespace: cap-app
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 1Gi
    - name: create deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: web
            namespace: cap-app
          spec:
            replicas: 2
            selector:
              matchLabels:
                app: web
            template:
              metadata:
                labels:
                  app: web
              spec:
                containers:
                  - name: web
                    image: nginx:1.24
EOF
oc login -u kubeadmin -p <pw> https://api.crc.testing:6443
ansible-playbook deploy.yml
```

- Verification (resources present):

```sh
oc get ns cap-app
oc get pvc -n cap-app
oc get deployment web -n cap-app
```

### Exercise 5 — RHCOA walkthrough

- Task: if you have access to a devstack/TripleO-style cloud, walk the full
  lifecycle; otherwise complete the knowledge bank and note the theory-only
  steps.
- Expected outcome: a project, an instance, a volume, and a floating IP all
  exist and the instance is reachable.

```sh
# Requires an OpenStack environment (RC file/clouds.yaml):
openstack project create labcap
openstack keypair create lab-key
openstack server create --flavor m1.small --image rocky-9.3 \
  --key-name lab-key --project labcap labvm
openstack volume create --size 5 labvol
openstack server add volume labvm labvol
openstack floating ip create external
openstack server add floating ip labvm <ip>
openstack security group rule create --ingress --protocol tcp \
  --dst-port 22 --remote-ip 0.0.0.0/0 default
ssh -i lab-key cloud-user@<ip>
```

- Verification (instance ACTIVE, volume attached, SSH works):

```sh
openstack server list
openstack volume list
ssh -i lab-key cloud-user@<ip> hostname
```

## Test yourself

When you can do Exercises 1–5 without the book, sit the capstone attempts:

- **Bank**: `banks/rhce` — **Training** on `playbooks` and
  `system-configuration`; then **Mastery** on `roles` and
  `automation-controller`; then full-bank **Exam** until ≥ 0.70 three times.
- **Bank**: `banks/rhca` — **Training** on `enterprise-architecture` and
  `security-architecture`; then **Mastery** on `openshift-architecture` and
  `automation-at-scale`; then full-bank **Exam** until ≥ 0.70 three times.
- **Bank**: `banks/rhcoa` — **Training** on `openstack-core` and
  `identity-keystone`; then **Mastery** on `compute-nova` and
  `networking-neutron`; then full-bank **Exam** until ≥ 0.70 three times.
- Treat a domain below 70% as a backlog: one focused Mastery attempt plus the
  matching exercise above, then re-sit the full bank.

## Self-check quiz

1. **Q**: Why is `ansible.builtin.dnf` preferred over `shell: dnf install -y`
   in an RHCE playbook? **A**: The module checks current state and is
   idempotent — it only installs when missing and reports `ok` when present;
   `shell` reruns unconditionally.
2. **Q**: An architect must meet RPO ≤ 1 minute between two sites. What
   drives the design? **A**: Synchronous (or near-sync) replication of the
   data plane, because RPO bounds acceptable data loss at failover.
3. **Q**: Why are control plane node counts odd in an HA design? **A**:
   Quorum — etcd requires a majority to be available, and an odd count of 3 or
   5 avoids tie votes while minimizing infrastructure.
4. **Q**: Which OpenStack service owns a floating IP? **A**: Neutron — a
   floating IP is an address on an external network associated with an
   instance port; Nova manages the instance itself.
5. **Q**: What does an OAuth identity provider do in OpenShift, and what
   comes after it? **A**: It authenticates users at the platform edge (OIDC
   or LDAP); after authentication, RBAC bindings map identities and groups to
   roles.

## See also

- RHCE, RHCA, and RHCOA certification pages on redhat.com (referenced by
  name; objectives change between releases).
- docs.ansible.com and docs.openstack.org for reference; the Ansible and
  OpenStack documentation portals.
- `ansible-doc kubernetes.core.k8s`, `openstack --help`.
- [Curriculum home](README.md) — the weekly plan and simulator-mode guide;
  from here, re-run any module whose domain scored below 70%.
