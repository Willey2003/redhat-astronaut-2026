# Module 5 — Automation and data storage (EX430 · EX370)

Module 5 is where the platform starts running itself. EX430 is the
certification for developing automation with Ansible and Red Hat Ansible
Automation Platform (AAP). EX370 is the OpenShift Data Foundation (ODF)
specialist certification — block/file/object storage for the cluster plus
backup and recovery. Together they give you the two force multipliers every
production platform needs: **automation** (make change repeatable) and
**storage** (make data durable). You need your cluster, a RHEL machine for
Ansible, and `ansible-core` installed.

## Learning objectives

1. Write correct, idempotent playbooks with the right modules for system
   configuration, and run them against RHEL hosts.
2. Structure automation in roles and collections, and consume content from
   Ansible Galaxy or a private hub.
3. Understand and operate AAP: projects, inventories, credentials, Job
   Templates, and Execution Environments.
4. Explain OpenShift Data Foundation: architecture, components, and
   installation of the ODF Operator.
5. Provision and consume persistent storage via ODF StorageClasses
   (RBD block, CephFS file), and configure object storage.
6. Back up and restore cluster and application data, and verify recovery.

## Key concepts

- **Inventory**: the list of managed hosts. `ansible -i hosts all -m ping`;
   groups and `[group:vars]` blocks. The `ansible` inventory (INI or YAML) is
   the authoritative source for what Ansible can manage.
- **Playbooks**: YAML documents of plays (`hosts`, `become`, `tasks`).
   `ansible-playbook -i hosts site.yml`. Modules are the unit of work; a good
   playbook is idempotent — running it twice changes nothing the second time.
- **Key modules**: `ansible.builtin.dnf` (packages), `ansible.builtin.service`
   /`systemd` (services), `ansible.builtin.copy`/`template` (files and
   Jinja2 rendering), `ansible.builtin.user`/`group`, `ansible.posix.firewalld`
   (ports), `ansible.builtin.lineinfile`.
- **Handlers**: run once at the end of the play when a task reports `changed`
   and `notify`s them — the standard way to restart a service after a config
   change.
- **Variables and facts**: `ansible_facts` gathered by the setup module
   (`ansible_facts['distribution']`); precedence from low to high — role
   defaults, inventory, play vars, `--extra-vars` wins.
- **Conditionals and loops**: `when:` gates tasks; `loop:`/`with_items`
   repeat them, exposing the current element as `item`.
- **Roles**: `ansible-galaxy role init myrole` produces `tasks/`,
   `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`. Reusable units
   invoked with the `roles:` keyword.
- **Collections**: namespaced bundles of modules, roles, and plugins —
   `ansible-galaxy collection install community.general`. Content shared via
   Galaxy or a private Automation Hub.
- **AAP architecture**: the **automation controller** schedules and logs job
   runs; **Execution Environments** are the container images that provide
   Ansible + Python + collections for each job; **Automation Hub** distributes
   validated content.
- **Controller objects**: `Project` (syncs from git), `Inventory`, machine
   `Credential`s (encrypted), and `Job Template` (binds playbook + inventory +
   credentials). Launching a Job Template creates a Job with full history.
- **ODF architecture**: ODF is built on Ceph + Rook. Components: Ceph cluster,
   Rook operator, and the OCS/ODF operator; provides `ocs-storagecluster-ceph-rbd`
   (block) and `-cephfs` (file) StorageClasses plus object (S3) storage via
   NooBaa/RGW.
- **Persistent storage with ODF**: `oc get storageclass`, PVCs bind
   dynamically, `oc set volumes deployment/db --add --claim-name=data`.
   StorageClass reclaim policy and access modes still apply — ODF just makes
   them self-service.
- **Backup and recovery**: application-level backups with a tool like OADP
   (Velero-based) targeting object storage; `restic` for file backups.
   `oc get backups`, `oc get restores`. Verify restores in a scratch project.

## Hands-on exercises

EX430 exercises run on a RHEL host (or your cluster's nodes if you are
brave). EX370 exercises run on your cluster with the ODF operator installed
(CRC can run ODF with enough resources, otherwise note which checks are
theory).

### Exercise 1 — First playbook and ad-hoc commands

- Task: write an inventory with one group, run an ad-hoc ping and a dnf
  install, then a two-task playbook.
- Expected outcome: `ansible` reports success; the playbook is idempotent
  (second run shows no `changed`).

```sh
mkdir -p ~/lab5 && cd ~/lab5
cat > hosts <<'EOF'
[web]
node1.example.com ansible_user=student
[web:vars]
ansible_become=yes
EOF
ansible -i hosts web -m ping
ansible -i hosts web -m dnf -a "name=postfix state=present"
cat > site.yml <<'EOF'
- name: Ensure postfix is present and running
  hosts: web
  tasks:
    - name: install postfix
      ansible.builtin.dnf:
        name: postfix
        state: present
    - name: start postfix
      ansible.builtin.service:
        name: postfix
        state: started
        enabled: yes
EOF
ansible-playbook -i hosts site.yml
ansible-playbook -i hosts site.yml
```

- Verification (second run reports `ok=3` / nothing changed):

```sh
ansible-playbook -i hosts site.yml | tail -2
```

### Exercise 2 — Templates, handlers, and variables

- Task: generate a config file from a template with a variable, and notify a
  handler that restarts the service on change.
- Expected outcome: the file contains the rendered variable; a second run does
  not restart the service.

```sh
mkdir -p templates group_vars
echo 'port: 8080' > group_vars/web.yml
cat > templates/app.conf.j2 <<'EOF'
listen_port={{ port }}
host={{ ansible_facts['hostname'] }}
EOF
cat > site2.yml <<'EOF'
- name: Render config with handler
  hosts: web
  handlers:
    - name: restart app
      ansible.builtin.service:
        name: postfix
        state: restarted
  tasks:
    - name: write config
      ansible.builtin.template:
        src: templates/app.conf.j2
        dest: /etc/app/app.conf
      notify: restart app
EOF
ansible-playbook -i hosts site2.yml
ansible-playbook -i hosts site2.yml
cat /etc/app/app.conf
```

- Verification (file exists with `listen_port=8080`, second run skips the
  handler):

```sh
cat /etc/app/app.conf
ansible-playbook -i hosts site2.yml | grep -c changed
```

### Exercise 3 — Roles and the automation controller

- Task: generate a role with `ansible-galaxy role init`, use it from a
  playbook, then describe the controller objects a Job Template needs.
- Expected outcome: the role's tasks run via the `roles:` keyword; you can
  name the controller objects (Project, Inventory, Credential, Job Template).

```sh
ansible-galaxy role init myweb
cat > myweb/tasks/main.yml <<'EOF'
- name: install httpd
  ansible.builtin.dnf:
    name: httpd
    state: present
- name: start httpd
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: yes
EOF
cat > site3.yml <<'EOF'
- name: Apply role
  hosts: web
  roles:
    - myweb
EOF
ansible-playbook -i hosts site3.yml
ansible-galaxy collection install community.general
```

- Verification (role ran, httpd active):

```sh
ansible-playbook -i hosts site3.yml | grep -c myweb
```

### Exercise 4 — ODF StorageClasses and a PVC

- Task: install the ODF operator (via OperatorHub), create a
  StorageCluster, and bind a PVC using the RBD StorageClass.
- Expected outcome: `oc get storageclass` shows the ODF classes; a PVC
  becomes `Bound` and can be mounted.

```sh
oc new-project lab-odf
oc create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: lab-odf
spec:
  accessModes: ["ReadWriteOnce"]
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-ceph-rbd
EOF
oc get pvc app-data -n lab-odf
oc get storageclass
```

- Verification (PVC Bound, storage class listed):

```sh
oc get pvc app-data -n lab-odf -o jsonpath='{.status.phase}{"\n"}'
oc get storageclass | grep ocs
```

### Exercise 5 — Backup and restore an application

- Task: install OADP (Velero-based), create a backup of a namespace, delete
  the application, then restore it.
- Expected outcome: the Backup shows `Completed`; after a restore the
  application's pods and PVCs come back.

```sh
oc new-project lab-backup
oc create deployment app --image=nginx:1.24 --replicas=2
# install OADP operator + create a DataProtectionApplication pointing at S3
oc create -f - <<'EOF'
apiVersion: oadp.openshift.io/v1alpha1
kind: Backup
metadata:
  name: app-backup
  namespace: openshift-adp
spec:
  includedNamespaces: ["lab-backup"]
  storageLocation: default
EOF
oc get backup app-backup -n openshift-adp
oc delete project lab-backup
# recreate project, create Restore, watch it complete
oc get restore -n openshift-adp
```

- Verification (backup Completed, restore done, pods back):

```sh
oc get backup app-backup -n openshift-adp -o jsonpath='{.status.phase}{"\n"}'
oc get pods -n lab-backup
```

## Test yourself

When you can do Exercises 1–5 without the book, sit simulator attempts:

- **Bank**: `banks/ex430-ansible` — **Training** on `playbooks-templating`
  and `roles-collections`, then **Mastery** on `automation-controller`.
- **Bank**: `banks/ex370-storage` — **Training** on
  `openshift-data-foundation` and `persistent-storage`, then **Mastery** on
  `backup-recovery`.
- Aim for 80%+ on the Mastery passes before starting Module 6; below that,
  redo Exercises 1–2 (Ansible mechanics) and 4–5 (storage mechanics).

## Self-check quiz

1. **Q**: What makes a playbook idempotent? **A**: Each module checks current
   state and only changes what differs, so running the same playbook twice
   produces the same end state with no unnecessary changes.
2. **Q**: When does a handler actually run? **A**: Once, at the end of the
   play, and only if a task that reported `changed` notified it.
3. **Q**: What four objects does a Job Template bind together? **A**: A
   Project (playbook source), an Inventory, Credentials, and the execution
   settings/Execution Environment.
4. **Q**: Which ODF StorageClasses map to block and file storage? **A**:
   `ocs-storagecluster-ceph-rbd` for RWO block and
   `ocs-storagecluster-cephfs` for RWX file; object (S3) comes from the
   NooBaa/RGW gateway.
5. **Q**: How do you prove a backup works? **A**: Restore it — OADP `Restore`
   into a scratch project and confirm the workloads and PVCs return.

## See also

- EX430 and EX370 certification pages on redhat.com (referenced by name).
- docs.ansible.com and docs.redhat.com for AAP; docs.openshift.com for ODF
  and OADP chapters.
- `ansible-doc <module>`, `oc explain storageclass`, `oc explain backup`.
- [Module 6 — Capstones: RHCE, RHCA, RHCOA](06-capstones.md) — next: prove
  the whole journey with the architect and engineer certifications.
