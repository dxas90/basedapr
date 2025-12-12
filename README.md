# GitOps Kubernetes Infrastructure with FluxCD

GitOps-based Kubernetes cluster management using FluxCD, Kustomize, Helm, and SOPS encryption.

## Overview

- **GitOps**: Declarative cluster state management via Git
- **FluxCD**: Automated reconciliation and continuous delivery
- **Kustomize**: Configuration overlays and customizations
- **Helm**: Application package management
- **SOPS**: Secret encryption with age keys
- **Multi-cluster**: Structured support for k3d, kind, AWS, Azure, GCP, k0s, metal (k3d fully implemented)

## Repository Structure

```
├── apps/                   # Application deployments
│   └── playground/        # Learning apps namespace
├── bootstrap/             # FluxCD installation
├── clusters/              # Cluster-specific configurations
│   └── k3d/              # k3d cluster (fully configured)
│       └── apps/         # MetalLB, local-path storage
├── common/                # Shared infrastructure
│   ├── dapr-system/      # Dapr, Dex, OpenFGA
│   ├── default/          # Default namespace apps
│   ├── external-secrets/ # External Secrets Operator
│   ├── istio-system/     # Istio service mesh
│   ├── keda/             # Event-driven autoscaling
│   ├── kube-system/      # Metrics server, Reloader
│   └── kube-tools/       # Kyverno policy engine
├── components/            # Kustomize components
└── operators/             # Percona operator

## Application Architecture

Each application follows this structure:

```text
app-name/
├── app/
│   ├── kustomization.yaml          # Lists all resources
│   ├── helmrepository.yaml         # Helm chart source (or ocirepository.yaml for OCI)
│   ├── helmrelease.yaml            # HelmRelease configuration
│   └── [additional-resources].yaml # ConfigMaps, Secrets, RBAC
└── install.yaml                    # FluxCD Kustomization pointing to ./common/<category>/app-name/app
```

## Infrastructure Components

**Currently Implemented:**

- **dapr-system**: Dapr runtime, Dex (OIDC), OpenFGA (authorization)
- **external-secrets**: Kubernetes External Secrets Operator
- **istio-system**: Istio service mesh components
- **keda**: Event-driven autoscaling
- **kube-system**: Metrics server, Reloader
- **kube-tools**: Kyverno policy engine
- **operators**: Percona database operator

**k3d Cluster Specific:**

- **MetalLB**: Load balancer for bare metal
- **local-path**: Local storage provisioner

## Quick Start

### Prerequisites

Tools managed via `mise.toml`:

```bash
mise install  # Installs flux2, kubectl, k3d, sops, age, dapr
```

### Bootstrap k3d Cluster

```bash
export CLUSTER=k3d

# 1. Apply FluxCD bootstrap
kubectl apply --kustomize bootstrap

# 2. Configure SOPS age key
cat ~/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin

# 3. Configure Git authentication
kubectl create secret generic flux-system \
  --namespace=flux-system \
  --from-file=identity=$HOME/.ssh/id_rsa \
  --from-file=identity.pub=$HOME/.ssh/id_rsa.pub \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null | grep -v '^#')"

# 4. Apply cluster configuration
kubectl apply -k clusters/${CLUSTER}
```

### Verify Deployment

```bash
flux get kustomizations
flux get helmreleases
flux logs --follow
```

## Adding New Applications

### 1. Create Application Structure

```bash
mkdir -p common/<category>/<app-name>/app
cd common/<category>/<app-name>
```

### 2. Add Helm Repository (HTTP charts)

```yaml
# app/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: app-name
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.example.com/
```

### 3. Or Add OCI Repository (OCI charts)

```yaml
# app/ocirepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: app-name
  namespace: flux-system
spec:
  interval: 1h
  url: oci://ghcr.io/example/helm/chart
  ref:
    tag: 1.0.0
```

### 4. Create HelmRelease

**For HTTP Helm repos:**

```yaml
# app/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: app-name
spec:
  interval: 30m
  chart:
    spec:
      chart: app-name
      sourceRef:
        kind: HelmRepository
        name: app-name
        namespace: flux-system
  values:
    # your values here
```

**For OCI repos:**

```yaml
# app/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: app-name
spec:
  interval: 30m
  chartRef:
    kind: OCIRepository
    name: app-name
  values:
    # your values here
```

### 5. Create App Kustomization

```yaml
# app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml  # or ocirepository.yaml
  - helmrelease.yaml
```

### 6. Create FluxCD Install Kustomization

```yaml
# install.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-name
  namespace: flux-system
spec:
  targetNamespace: target-namespace
  path: ./common/<category>/<app-name>/app
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  prune: true
  wait: false
  interval: 30m
```

### 7. Update Parent Kustomization

Add to `common/<category>/kustomization.yaml`:

```yaml
resources:
  - <app-name>/install.yaml
```

## Secret Management

### SOPS with age

```bash
# Encrypt
sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml

# Edit
sops secret.sops.yaml

# Decrypt
sops --decrypt secret.sops.yaml
```

### External Secrets Operator

Integrates with HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, Google Secret Manager.

## Multi-Cluster Support

Clusters: k3d (configured), kind, aws, azure, gcp, k0s, metal (placeholders)

Override common configs per cluster in `clusters/<type>/apps/`.

## Troubleshooting

```bash
# Reconcile FluxCD
flux reconcile kustomization flux-system
flux logs --follow

# Check HelmReleases
flux get helmreleases
kubectl describe helmrelease <name> -n <namespace>

# Debug SOPS
kubectl logs -n flux-system deployment/kustomize-controller
```

## References

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [SOPS Documentation](https://github.com/mozilla/sops)
