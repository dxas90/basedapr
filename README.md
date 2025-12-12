# GitOps Kubernetes Infrastructure with FluxCD

This repository implements a comprehensive GitOps approach for managing Kubernetes clusters using FluxCD. It provides a structured, scalable solution for deploying and managing infrastructure applications, observability tools, and cluster configurations across multiple environments.

## 🏗️ Project Overview

This project demonstrates modern Kubernetes cluster management using:

- **GitOps Principles**: All cluster state is declaratively managed through Git
- **FluxCD**: Continuous delivery and reconciliation of cluster resources
- **Kustomize**: Configuration management and environment-specific customizations
- **Helm**: Package management for complex applications
- **SOPS**: Secret encryption and management
- **Multi-cluster Support**: Organized structure for multiple cluster types (k3d, kind, AWS, Azure, GCP, etc.)

## 📁 Repository Structure

```
├── bootstrap/              # FluxCD bootstrap configuration
├── clusters/               # Cluster-specific configurations
│   ├── k3d/               # Local k3d cluster configuration
│   ├── kind/              # Kind cluster configuration
│   ├── aws/               # AWS EKS cluster configuration
│   ├── azure/             # Azure AKS cluster configuration
│   └── gcp/               # Google GKE cluster configuration
├── common/                 # Shared infrastructure applications
│   ├── cert-manager/      # Certificate management
│   ├── external-secrets/  # External secret management
│   ├── kube-system/       # Core Kubernetes system apps
│   ├── kube-tools/        # Kubernetes tooling (Kyverno, NFD)
│   ├── monitoring/        # Infrastructure monitoring stack
│   ├── networking/        # Network management (Traefik, Gateway API)
│   ├── observability/     # Observability platform (Grafana, Loki, etc.)
│   ├── dbms/              # Database management systems
│   └── vault/             # HashiCorp Vault
└── repositories/           # Legacy repository definitions (being phased out)
```

## 🎯 Key Features

### Application Architecture
Each application follows a consistent pattern:
```
app-name/
├── app/
│   ├── kustomization.yaml      # App resources configuration
│   ├── helmrepository.yaml     # Helm repository definition
│   ├── helmrelease.yaml        # Application deployment
│   └── [other-resources].yaml  # ConfigMaps, Secrets, etc.
└── install.yaml                # FluxCD Kustomization
```

### Infrastructure Components

#### **Core Infrastructure** (`common/`)
- **cert-manager**: Automatic TLS certificate management
- **external-secrets**: Kubernetes External Secrets Operator
- **vault**: HashiCorp Vault for secret management

#### **Kubernetes System** (`common/kube-system/`)
- **cilium**: CNI and network security
- **coredns**: DNS service
- **csi-driver-nfs**: NFS storage driver
- **metrics-server**: Resource metrics collection

#### **Kubernetes Tools** (`common/kube-tools/`)
- **kyverno**: Policy engine for Kubernetes
- **node-feature-discovery**: Hardware feature detection

#### **Monitoring Stack** (`common/monitoring/`)
- **kube-prometheus-stack**: Prometheus, Grafana, AlertManager
- **gatus**: Service health dashboard
- **goldilocks**: Resource recommendation
- **vpa**: Vertical Pod Autoscaler

#### **Observability Platform** (`common/observability/`)
- **grafana**: Visualization and dashboards
- **loki**: Log aggregation
- **alloy**: Telemetry collection
- **victoria-logs**: High-performance log storage
- **alertmanager**: Alert routing and management

#### **Networking** (`common/networking/`)
- **traefik**: Ingress controller and load balancer
- **gateway-api**: Next-generation ingress APIs

#### **Database Systems** (`common/dbms/`)
- **cloudnative-pg**: PostgreSQL operator
- **cockroachdb**: Distributed SQL database
- **nats**: Message streaming
- **dragonfly**: Redis-compatible in-memory store

## 🚀 Quick Start

### Prerequisites
- A running Kubernetes cluster
- `kubectl` installed and configured
- `flux` CLI installed
- Git repository access
- SSH key pair for authentication
- `age` key for SOPS encryption (optional)

### 1. Bootstrap FluxCD

Apply the FluxCD bootstrap configuration:

```bash
kubectl apply --kustomize bootstrap
```

This installs FluxCD components in the `flux-system` namespace.

### 2. Configure Git Authentication

Create a Kubernetes secret for Git repository access:

```bash
kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=identity=<PRIVATE_KEY_PATH> \
    --from-file=identity.pub=<PUBLIC_KEY_PATH> \
    --from-literal=known_hosts="$(ssh-keyscan -p <PORT|22> <GIT_REPO_HOST> 2>/dev/null | grep -v '^#')"
```

### 3. Deploy Cluster Configuration

Choose your cluster type and apply the configuration:

```bash
# For k3d local development
kubectl apply --kustomize clusters/k3d

# For production AWS EKS
kubectl apply --kustomize clusters/aws

# For production Azure AKS
kubectl apply --kustomize clusters/azure
```

### 4. Verify Deployment

Check FluxCD reconciliation status:

```bash
flux get kustomizations
flux get helmreleases
```

## 🔧 Configuration Management

### Environment Variables
Cluster-specific variables are managed in:
- `clusters/<cluster-type>/vars/cluster-settings.yaml`
- `clusters/<cluster-type>/vars/cluster-secrets.yaml` (SOPS encrypted)

### Adding New Applications

1. **Create Application Structure**:
   ```bash
   mkdir -p common/new-app/app
   cd common/new-app
   ```

2. **Add Repository Definition**:
   ```yaml
   # app/helmrepository.yaml (for Helm charts)
   apiVersion: source.toolkit.fluxcd.io/v1
   kind: HelmRepository
   metadata:
     name: new-app
     namespace: flux-system
   spec:
     interval: 1h
     url: https://charts.example.com/
   ```

3. **Create HelmRelease**:
   ```yaml
   # app/helmrelease.yaml
   apiVersion: helm.toolkit.fluxcd.io/v2
   kind: HelmRelease
   metadata:
     name: new-app
   spec:
     interval: 30m
     chart:
       spec:
         chart: new-app
         sourceRef:
           kind: HelmRepository
           name: new-app
           namespace: flux-system
   ```

4. **Create Kustomization**:
   ```yaml
   # app/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - helmrepository.yaml
     - helmrelease.yaml
   ```

5. **Create Install Configuration**:
   ```yaml
   # install.yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: new-app
   spec:
     targetNamespace: new-app
     path: ./kubernetes/main/apps/new-app/app
     sourceRef:
       kind: GitRepository
       name: flux-system
       namespace: flux-system
     prune: true
     wait: false
     interval: 30m
   ```

6. **Update Parent Kustomization**:
   Add to `common/kustomization.yaml`:
   ```yaml
   resources:
     - new-app/install.yaml
   ```

## 🔐 Secret Management

### SOPS Integration
Secrets are encrypted using SOPS with age keys:

```bash
# Encrypt a secret
sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml

# Edit encrypted secret
sops secret.sops.yaml

# Decrypt for viewing
sops --decrypt secret.sops.yaml
```

### External Secrets
The External Secrets Operator integrates with:
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

## 🏢 Multi-Cluster Management

### Cluster Types Supported:
- **k3d**: Local development with k3d
- **kind**: Local development with kind
- **aws**: Amazon EKS production clusters
- **azure**: Azure AKS production clusters
- **gcp**: Google GKE production clusters
- **k0s**: Bare metal k0s clusters
- **metal**: Physical server deployments

### Cluster-Specific Customizations
Each cluster can override common configurations using Kustomize:

```yaml
# clusters/aws/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../common
patchesStrategicMerge:
  - aws-specific-overrides.yaml
```

## 📊 Monitoring and Observability

### Access Dashboards
- **Grafana**: `https://grafana.<your-domain>`
- **Prometheus**: `https://prometheus.<your-domain>`
- **AlertManager**: `https://alertmanager.<your-domain>`
- **Gatus**: `https://status.<your-domain>`

### Key Metrics Monitored:
- Cluster resource utilization
- Application health and performance
- Network traffic and security
- Storage and database metrics
- Custom business metrics

## 🔄 GitOps Workflow

1. **Make Changes**: Update configurations in Git
2. **Commit & Push**: Push changes to the repository
3. **FluxCD Sync**: FluxCD automatically detects and applies changes
4. **Verification**: Monitor deployment status through FluxCD
5. **Rollback**: Use Git history for quick rollbacks if needed

## 🚨 Troubleshooting

### Common Issues

**FluxCD not syncing:**
```bash
flux reconcile kustomization flux-system
flux logs --follow
```

**Application deployment failed:**
```bash
flux get helmreleases
kubectl describe helmrelease <app-name>
```

**Secret decryption issues:**
```bash
kubectl logs -n flux-system deployment/kustomize-controller
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes following the established patterns
4. Test in a local k3d cluster
5. Submit a pull request

## 📚 References

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [SOPS Documentation](https://github.com/mozilla/sops)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Step 2: Create Flux System Secret

Create a Kubernetes secret to allow Flux to authenticate with your Git repository:

```sh
kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=identity=<PRIVATE_KEY_PATH> \
    --from-file=identity.pub=<PUBLIC_KEY_PATH> \
    --from-literal=known_hosts="$(ssh-keyscan -p <PORT|22> <GIT_REPO_HOST> 2>/dev/null | grep -v '^#')"

kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=githubAppPrivateKey=gihub-app-key.pem \
    --from-literal=githubAppInstallationID="${FLUX_GITHUB_APP_INSTALLATION_ID}" \
    --from-literal=githubAppID="${FLUX_GITHUB_APP_ID}"

  flux create secret githubapp flux-system --app-id=${FLUX_GITHUB_APP_ID} --app-installation-id=${FLUX_GITHUB_APP_INSTALLATION_ID} --app-private-key=gihub-app-key.pem
```

Replace:
- `<PRIVATE_KEY_PATH>` with the path to your private SSH key
- `<PUBLIC_KEY_PATH>` with the path to your public SSH key
- `<PORT|22>` with the SSH port of your Git server (default: `22`)
- `<GIT_REPO_HOST>` with the hostname of your Git repository server

---

## Step 3: Create SOPS Age Secret (Optional)

If you are using SOPS for secret encryption, store the Age key as a Kubernetes secret:

```sh
cat <YOUR_AGE_KEY_FILE> | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin
```

Replace `<YOUR_AGE_KEY_FILE>` with the path to your Age key file.

---

## Next Steps

Once these steps are completed:
- Verify Flux components are running:
  ```sh
  kubectl get pods -n flux-system
  ```
- Check the Flux logs for any errors:
  ```sh
  kubectl logs -n flux-system deployment/flux-system
  ```
- Confirm that the cluster is syncing with your Git repository.

For further configurations, refer to the [Flux CD documentation](https://fluxcd.io/docs/).

---

## Repository Structure

### `apps/`
Contains application manifests managed using Kustomize. Applications are categorized into different environments:
- `base/` - Base configuration for all applications.
- `development/`, `staging/`, `production/`, `testing/` - Environment-specific overlays for applications.

### `bootstrap/`
Contains bootstrapping configurations for initializing FluxCD on a cluster.

### `clusters/`
Defines the Kubernetes clusters managed by FluxCD, including:
- `docker-desktop-kind/`
- `k3d/`
- `production/`, `staging/`, `testing/`

Each cluster includes:
- `flux-system/` - FluxCD system manifests.
- `vars/` - Cluster-specific settings and secrets.

### `infrastructure/`
Manages core infrastructure components deployed to Kubernetes:
- `base/` - Common infrastructure components.
- `development/`, `staging/`, `production/`, `testing/` - Environment-specific overlays.

Includes HelmReleases for managing:
- Networking (Traefik, Nginx, MetalLB, Cilium)
- Monitoring (Prometheus, Grafana, Gatus, Goldilocks)
- Security (Kyverno, External Secrets, Cert-Manager)

### `repositories/`
Defines external dependencies, such as Git and Helm repositories:
- `git/` - External Git repositories.
- `helm/` - Helm repositories for third-party applications.
- `oci/` - OCI-based Flux manifests.

## FluxCD Workflow
1. **Bootstrap FluxCD**: Apply configurations in `bootstrap/` to set up Flux on a cluster.
2. **Manage Clusters**: Define cluster-specific configurations in `clusters/`.
3. **Deploy Applications**: Update `apps/` to manage application releases.
4. **Manage Infrastructure**: Update `infrastructure/` for Kubernetes platform services.
5. **Sync with FluxCD**: Flux continuously reconciles cluster state with this repository.

## Contributing
- Ensure all changes are reviewed before merging.
- Follow GitOps best practices for managing resources.
- Use Kustomize overlays to manage environment-specific settings.

## License
This repository is licensed under the MIT License.


## Troubleshooting

- If Flux fails to authenticate with the Git repository, verify the SSH keys and `known_hosts` configuration.
- Check the `flux-system` logs for errors related to Git authentication.
- Ensure your cluster has internet access if pulling manifests from an external source.

---

## References

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [SOPS for Kubernetes](https://fluxcd.io/flux/guides/mozilla-sops/)


### General setup

```shell
  export CLUSTER=kind
  kubectl apply --kustomize bootstrap ; \
  cat $HOME/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin ; \
  kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=identity=$HOME/Vaults/Safe/flux/fleet-flux-github \
    --from-file=identity.pub=$HOME/Vaults/Safe/flux/fleet-flux-github.pub \
    --from-literal=known_hosts="$(ssh-keyscan -p 4022 gitea.dxas90.work 2>/dev/null | grep -v '^#')" ; \
  sops --decrypt clusters/${CLUSTER}/vars/cluster-secrets.sops.yaml | kubectl apply -f - ; \
  envsubst < clusters/${CLUSTER}/vars/cluster-settings.yaml | kubectl apply -f - ; \
  kubectl apply -k clusters/${CLUSTER}
```
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: clusterip
  ingress_type: ingress
  ingress_hosts:
    - hostname: awx-demo.example.com
  ingress_annotations: |
    environment: testing
