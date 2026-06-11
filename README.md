# 🧩 Ansible Collection — `mb_wali.platform_core`

A production-ready Ansible collection for deploying and managing core Kubernetes platform services using **Helm**.

It provides a standardized, repeatable way to install and operate essential platform components across Kubernetes clusters.

---

## 🚀 Overview

`platform_core` is designed to simplify and standardize the lifecycle management of foundational Kubernetes services, including:

- 🌐 Cilium (CNI / networking)
- 🔁 Argo CD (GitOps)
- 🛡️ Kyverno (policy enforcement)
- ☸️ Rancher (cluster management)
- 🚪 Ingress Controllers / Gateway API
- 📊 Observability stack *(optional extension)*

All components are deployed using **Helm via Ansible**, ensuring consistency, idempotency, and version control.

---

## ✨ Key Features

- 📦 Helm-based deployment of Kubernetes platform services  
- 🔁 Fully idempotent install and upgrade workflows  
- 🧱 Modular, role-based architecture  
- 📌 Version-pinned deployments for stability and reproducibility  
- 🧑‍🔧 Built for platform engineering teams and SREs  
- 🔧 Easily extensible for additional platform components  

---

## 🗂️ Collection Structure
```bash
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

Before using this collection, ensure the following prerequisites are met:

- Ansible ≥ 2.15  
- `kubernetes.core` collection  
- Access to a Kubernetes cluster (API key required from the cluster; no kubeconfig or direct control-node access needed)
- Helm installed in the execution environment  

### Install dependencies

```bash
ansible-galaxy collection install kubernetes.core
```

### 📦 Installation

**Install the collection**
```bash
ansible-galaxy collection install mb_wali.platform_core
```

---

## ▶️ Usage

### Example Playbook

Once installed, roles can be referenced using the fully qualified collection name (FQCN):

```yaml
- name: Deploy Kubernetes platform core services
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

## 🔐 Kubernetes Authentication

Set the required environment variables for cluster access:
```bash
export K8S_AUTH_HOST=https://my-cluster:6443
export K8S_AUTH_API_KEY=<token>
export K8S_AUTH_VERIFY_SSL=false
```

---

## 🏗️ Build the Collection

To package the collection locally:
```bash
ansible-galaxy collection build
```
