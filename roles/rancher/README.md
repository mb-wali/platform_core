# [Rancher](https://github.com/rancher/rancher)
**Current Version: v2.14**

Deploying Rancher with `ingress.enabled=false` which means its runnig fully `http` in-cluster.

You can get the generated password run: ` kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}' `

[Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version)

[Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions) - To check compatabilities.

[Uninstall Rancher](https://github.com/rancher/rancher-cleanup/tree/main)

---

## Upgrading Rancher
Befor upgrading Rancher have a look at supported versions of kubernetes: [Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions)

**in this order..**

1. Upgrade the K3S Cluster
2. Upgrade the Rancher helm chart version.

---

## OIDC 
Comingsoon...
