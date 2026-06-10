
# Ansible Collection - mb_wali.platform_core

Ansible collection for deploying and managing core Kubernetes platform services using Helm.

This collection provides lifecycle management for essential platform components such as networking, security, GitOps, and policy enforcement in Kubernetes clusters.

---

## 🚀 What this collection does

`platform_core` standardizes the deployment and management of:

- Cilium (CNI / networking)
- Argo CD (GitOps)
- Kyverno (policy engine)
- Rancher (cluster management)
- Ingress controllers / Gateway API
- Observability components (optional extension)

All components are deployed and managed using **Helm via Ansible**.

---

## 🧱 Key Features

- Helm-based deployment of Kubernetes platform services
- Idempotent install and upgrade workflows
- Modular role-based architecture
- Version-pinned deployments for stability
- Designed for platform engineering teams and SREs

---

## 📦 Collection Structure

```
platform_core/
├── galaxy.yml
├── roles/
│ ├── cilium/
│ ├── argocd/
│ ├── kyverno/
│ └── ...
├── playbooks/
│ └── platform.yml
└── docs/
```

---

## ⚙️ Requirements

- Ansible >= 2.15
- kubernetes.core collection
- Access to a Kubernetes cluster
- Helm installed on execution environment

Install dependencies:

```bash
ansible-galaxy collection install kubernetes.core
```

---

## ▶️ Usage

**Install the collection**

```bash
ansible-galaxy collection install mb_wali.platform_core
```


**Use in a playbook**

Once installed, you reference roles using the fully qualified collection name (FQCN):

```yaml
- name: Deploy platform core services
  hosts: localhost
  gather_facts: false

  collections:
    - mb_wali.platform_core

  roles:
    - role: cilium
      vars:
        cilium_helm_version: "1.18.3"
        cilium_kubeproxy_replacement: "false"
```

---

**Build Collection**

```bash
ansible-galaxy collection build
```
