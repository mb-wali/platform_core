# 🧩 Ansible Collection — `mb_wali.platform_core`

A production-ready Ansible collection for deploying and managing core Kubernetes platform services using **Helm**.

It provides a standardized, repeatable way to install and operate essential platform components across Kubernetes clusters.

---

## 🚀 Overview

`platform_core` provides a standardized and automated approach for managing **top-level Kubernetes cluster configurations**, foundational platform services, and cluster lifecycle operations.

The Ansible Galaxy collection / roles included in this project are responsible for the **cluster-wide lifecycle management** of core Kubernetes components, ensuring consistent deployment patterns, repeatability, and operational reliability.

In addition to initial provisioning, `platform_core` is used to perform **controlled platform upgrades**, including Helm release updates, component version changes, and configuration migrations across Kubernetes clusters.

Managed platform components include:

- 🌐 **Cilium** — CNI, networking, and cluster connectivity
- 🔁 **Argo CD** — GitOps-based application delivery
- 🛡️ **Kyverno** — Kubernetes policy management and enforcement
- ☸️ **Rancher** — Cluster management and administration
- 💾 **Ceph CSI** — Persistent storage integration and dynamic volume provisioning
- 🚪 **Ingress Controllers / Gateway API** — Traffic management and external access
- 📊 **Observability Stack** *(optional extension)* — Monitoring, logging, and platform visibility

All components are deployed and configured using **Helm through Ansible**, providing:

- ✅ Consistent installation workflows
- 🔁 Idempotent cluster operations
- 📦 Version-controlled deployments
- 🔧 Declarative configuration management
- ⬆️ Automated and repeatable upgrade processes
- 🚀 Reliable platform lifecycle management

These roles are intended for **foundational cluster setup and maintenance only** and establish the base platform layer for higher-level application deployments.

---

## ✨ Key Features

- 📦 **Helm-based deployments** — Standardized installation and management of Kubernetes platform services
- 🔁 **Idempotent lifecycle operations** — Safe, repeatable provisioning, configuration, and upgrade workflows
- 🧱 **Modular role-based architecture** — Independent Ansible Galaxy roles for each platform component
- 📌 **Version-controlled deployments** — Pinned Helm chart and component versions for stability and reproducibility
- ⬆️ **Upgrade automation** — Simplified platform component upgrades with controlled version transitions
- 🔧 **Declarative configuration management** — Cluster state defined through Ansible variables and Helm values
- 🛡️ **Production-focused platform foundation** — Built for platform engineering teams, DevOps, and SRE operations
- 🧩 **Extensible design** — Easily integrate additional Kubernetes services and infrastructure components
- ☸️ **Cluster-wide configuration management** — Designed for foundational Kubernetes platform setup and maintenance

---

## ⚙️ Requirements

Before using this collection, ensure the following prerequisites are met:

- 🚀 **Ansible ≥ 2.15**
- 📦 **`kubernetes.core` Ansible collection**
- ☸️ **Access to a Kubernetes cluster**
  - A valid **Kubernetes API token** is required
  - No `kubeconfig` file is needed
  - No direct SSH access to cluster nodes is required
  - No control-plane or worker node access is required
- ⛵ **Helm installed** in the Ansible execution environment
- 🔐 **Network access to the Kubernetes API endpoint**

`platform_core` connects directly to the Kubernetes API using token-based authentication.

Configure the cluster connection using the following environment variables:

```bash
export K8S_AUTH_HOST=https://my-cluster:6443
export K8S_AUTH_API_KEY=<token>
export K8S_AUTH_VERIFY_SSL=false
```

Where:

* K8S_AUTH_HOST — Kubernetes API server endpoint
* K8S_AUTH_API_KEY — Kubernetes service account or API token with required permissions
* K8S_AUTH_VERIFY_SSL — TLS certificate verification setting (true recommended for production)

---

## 📦 Installation

Install the required dependencies and `platform_core` collection:

```bash
# Install dependencies
ansible-galaxy collection install kubernetes.core

# Install collection 
ansible-galaxy collection install mb_wali.platform_core
```


---

## ▶️ Usage

### Example Playbook

Once installed, roles can be referenced using the fully qualified collection name (FQCN):

```yaml
```yaml
- name: Deploy Kubernetes platform core services
  hosts: localhost
  gather_facts: false

  collections:
    - mb_wali.platform_core

  environment:
    K8S_AUTH_HOST: "https://my-cluster:6443"
    K8S_AUTH_API_KEY: "<token>"
    K8S_AUTH_VERIFY_SSL: "false"

  roles:
    - role: cilium
      vars:
        cilium_helm_version: "1.18.3"
        cilium_kubeproxy_replacement: "false"
    - role: rancher
    ...
```

---
