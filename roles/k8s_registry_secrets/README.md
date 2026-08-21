# k8s_registry_secrets

An Ansible role to create Kubernetes `imagePullSecrets` for private container registries using `kubernetes.core`.

The role supports:
- Multiple container registries
- Different credentials per registry
- Creating secrets in multiple Kubernetes namespaces
- Generic registry authentication (GitLab, Harbor, Docker Hub, private registries, etc.)

## Role Variables

### `k8s_registry_secrets`

List of Kubernetes registry secrets to create.

Default:

```yaml
k8s_registry_secrets: []
```
Each item represents one Kubernetes `imagePullSecret`.

Example:
```yaml
k8s_registry_secrets:
  - name: gitlab-registry # name of the secret
    registry_url: registry.gitlab.com
    username: gitlab-ci-token
    password: "{{ gitlab_token }}"
    namespaces:
      - app-dev
      - app-prod

  - name: harbor-registry
    registry_url: harbor.example.com
    username: robot$myproject
    password: "{{ harbor_robot_secret }}"
    namespaces:
      - app-dev
```
