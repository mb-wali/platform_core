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

---

## Kubernetes API Authentication

`platform_core` connects directly to the Kubernetes API using the authentication mechanism provided by the Ansible `kubernetes.core` collection and the Python Kubernetes client.

Instead of using a kubeconfig file, the cluster connection can be configured through environment variables:

```bash
export K8S_AUTH_HOST=https://my-cluster:6443
export K8S_AUTH_API_KEY=<token>
export K8S_AUTH_VERIFY_SSL=false
```

### How it works

The `kubernetes.core` Ansible modules (`k8s`, `k8s_info`, `helm`, `helm_repository`, etc.) use the Kubernetes Python client internally.

When a Kubernetes-related task runs, the client builds the Kubernetes API connection configuration by checking available authentication sources. The main sources are:

1. Module parameters provided directly in the Ansible task.
2. `K8S_AUTH_*` environment variables.
3. A kubeconfig file.
4. In-cluster Kubernetes configuration (when running inside Kubernetes).

In this setup, the environment variables provide all required connection details, so no kubeconfig file is required.

For example, this task:

```yaml
- name: Wait for contour envoy daemonset to be ready
  kubernetes.core.k8s_info:
    api_version: apps/v1
    kind: DaemonSet
    namespace: contour
    name: contour-envoy
```

automatically uses the Kubernetes API connection defined by:

```bash
K8S_AUTH_HOST
K8S_AUTH_API_KEY
K8S_AUTH_VERIFY_SSL
```

### Environment variables

| Variable | Description |
|---|---|
| `K8S_AUTH_HOST` | Kubernetes API server endpoint. Example: `https://my-cluster:6443` |
| `K8S_AUTH_API_KEY` | Kubernetes bearer token used for authentication. This is typically a ServiceAccount token or another Kubernetes API token with the required RBAC permissions. |
| `K8S_AUTH_VERIFY_SSL` | Controls TLS certificate verification. Set to `true` to verify the API server certificate. Set to `false` only when certificate verification is intentionally disabled. |

### Authentication flow

The connection flow is:

```
Ansible task
    |
    v
kubernetes.core module
    |
    v
Python Kubernetes client
    |
    +--> Reads K8S_AUTH_HOST
    |
    +--> Reads K8S_AUTH_API_KEY
    |
    +--> Reads K8S_AUTH_VERIFY_SSL
    |
    v
Authenticated request to Kubernetes API server
```

### Security considerations

The API token must have sufficient Kubernetes RBAC permissions for the operations performed by the role.

For example, installing Contour requires permissions to create and manage resources such as:

- Namespaces
- Deployments
- DaemonSets
- Services
- ConfigMaps
- ServiceAccounts
- Roles and RoleBindings
- Custom Resource Definitions (CRDs)

For production environments:

- Use a dedicated ServiceAccount with the minimum required permissions.
- Store the token securely using a secrets manager or Ansible Vault.
- Keep `K8S_AUTH_VERIFY_SSL=true` whenever possible.
- Avoid exposing API tokens in logs, CI output, or plain text configuration files.

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

## Build collection
```bash
ansible-galaxy collection build
```
