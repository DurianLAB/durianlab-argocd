# ArgoCD Ansible Deployment

Ansible playbook and Jenkins pipeline for deploying ArgoCD with GitOps support.

## Prerequisites

### Jenkins Setup

#### 1. Kubernetes RBAC

Create a `jenkins-agent` ServiceAccount with cluster admin permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-agent
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-agent-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: jenkins-agent
    namespace: jenkins
```

Apply:
```bash
kubectl apply -f jenkins-rbac.yaml
```

#### 2. Kubeconfig Secret

Create a secret containing the kubeconfig for target clusters:

```bash
# For diskless-k8s cluster
incus exec diskless:k8s-cp -- cat /etc/kubernetes/admin.conf | base64 > kubeconfig-base64.txt

kubectl create secret generic jenkins-kubeconfig \
  --from-file=config=/path/to/kubeconfig \
  --type=Opaque \
  --namespace=jenkins
```

#### 3. Ansible ConfigMap (optional)

If using custom Ansible configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ansible-config
  namespace: jenkins
data:
  ansible.cfg: |
    [defaults]
    inventory = inventories/dev
    roles_path = roles
    collections_path = collections
    retry_files_enabled = False
```

## Jenkins Pipeline Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ARGOCD_NAMESPACE` | `argocd` | Target namespace (use `argocd-test-${BUILD_NUMBER}` for ephemeral) |
| `ARGOCD_RELEASE_NAME` | `argocd` | Helm release name |
| `ARGOCD_CHART_VERSION` | `7.7.7` | ArgoCD Helm chart version |
| `ENABLE_ANONYMOUS_ACCESS` | `true` | Enable view-only access |
| `EPHEMERAL_CLEANUP` | `false` | Delete namespace after testing |
| `KUBECONFIG_SOURCE` | `diskless-k8s` | Cluster: `diskless-k8s` or `k3s-dev` |

## Usage

### Local Testing

```bash
# Install dependencies
pip install ansible ansible-lint kubernetes netaddr
ansible-galaxy collection install -r requirements.yml

# Run playbook
ansible-playbook playbooks/playbook-argocd.yml -i inventories/dev \
  -e "argocd_namespace=argocd-test" \
  -e "kubeconfig_file=/path/to/kubeconfig"
```

### Ephemeral Environment Testing

In Jenkins, set:
- `ARGOCD_NAMESPACE` = `argocd-test-${BUILD_NUMBER}`
- `EPHEMERAL_CLEANUP` = `true`

This will:
1. Deploy ArgoCD to a unique namespace
2. Run tests
3. Automatically clean up the namespace

## Repository Structure

```
.
├── ansible.cfg              # Ansible configuration
├── inventories/
│   └── dev                  # Development inventory
├── Jenkinsfile              # Jenkins pipeline
├── playbooks/
│   └── playbook-argocd.yml  # Main playbook
├── requirements.yml          # Ansible collection requirements
└── roles/
    └── argocd-deploy/       # ArgoCD deployment role
```

## Accessing ArgoCD

After deployment:
- **URL**: `http://<node-ip>:30080`
- **Anonymous Access**: Enabled by default (view-only)
- **Admin Access**: Login with initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## GitOps Application

The playbook creates an ArgoCD Application that syncs from:
- **Repo**: `https://github.com/DurianLAB/ansible.git`
- **Path**: `gitops`
- **Target**: `default` namespace

Create the `gitops/` directory in your repo with Kubernetes manifests to deploy via GitOps.
