# GitOps FluxCD Kubernetes Infrastructure

GitOps-based Kubernetes cluster management using FluxCD, Kustomize, Helm, and SOPS encryption.

## Architecture

**Four-Layer Structure:**

1. **`bootstrap/`** - FluxCD installation
2. **`apps/`** - Application deployments (playground namespace with learning apps)
3. **`clusters/<type>/`** - Cluster-specific config (k3d fully implemented; others are placeholders)
4. **`common/`** - Shared infrastructure components

**GitOps Flow:** Git commits → FluxCD detects changes → Kustomizations reconcile → HelmReleases deploy apps

**Current Infrastructure:**

- **dapr-system**: Dapr runtime, Dex (OIDC), OpenFGA
- **external-secrets**: Kubernetes External Secrets Operator
- **istio-system**: Istio service mesh (base, cni, istiod, ingressgateway, ztunnel)
- **keda**: Event-driven autoscaling
- **kube-system**: Metrics server, Reloader
- **kube-tools**: Kyverno policy engine
- **operators**: Percona database operator
- **k3d specific**: MetalLB load balancer, local-path storage

## Critical Workflows

### Bootstrap k3d Cluster

```bash
export CLUSTER=k3d

# 1. Bootstrap FluxCD
kubectl apply --kustomize bootstrap

# 2. Configure SOPS age key
cat ~/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin

# 3. Configure Git authentication
kubectl create secret generic flux-system --namespace=flux-system \
  --from-file=identity=$HOME/.ssh/id_rsa \
  --from-file=identity.pub=$HOME/.ssh/id_rsa.pub \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null | grep -v '^#')"

# 4. Apply cluster configuration
kubectl apply -k clusters/${CLUSTER}
```

### Check Status

```bash
flux get kustomizations
flux get helmreleases
flux logs --follow
```

### Tool Management

All tools managed via `mise.toml`: flux2, kubectl, k3d, sops, age, dapr, direnv, uv

```bash
mise install  # Install all tools
```

## Application Structure Pattern

**Standard structure:**

```text
app-name/
├── app/
│   ├── kustomization.yaml          # Lists all resources
│   ├── helmrepository.yaml         # Helm chart source (or ocirepository.yaml for OCI)
│   ├── helmrelease.yaml            # HelmRelease configuration
│   └── [additional-resources].yaml # ConfigMaps, Secrets, RBAC
└── install.yaml                    # FluxCD Kustomization pointing to ./common/<category>/<app-name>/app
```

**Adding New Apps:**

1. Create: `common/<category>/<app-name>/app/`
2. Add `helmrepository.yaml` (or `ocirepository.yaml`)
3. Add `helmrelease.yaml` with `chart.spec` (HTTP) or `chartRef` (OCI)
4. Create `app/kustomization.yaml` listing resources
5. Create `install.yaml` FluxCD Kustomization
6. Add `- <app-name>/install.yaml` to `common/<category>/kustomization.yaml`

## FluxCD Resource Conventions

**HelmRelease with OCI:**

```yaml
spec:
  chartRef:
    kind: OCIRepository
    name: chart-name
```

**HelmRelease with HTTP:**

```yaml
spec:
  chart:
    spec:
      chart: chart-name
      sourceRef:
        kind: HelmRepository
        name: repo-name
        namespace: flux-system
```

**FluxCD Kustomization:**

- `sourceRef.name: flux-system` (standard for this repo)
- `path`: Relative path (e.g., `./common/dapr-system/dapr/app`)
- `targetNamespace`: Deployment target
- `prune: true`, `wait: false` are defaults
- Use `dependsOn` for ordering (see MetalLB example)

## Apps Layer

**`apps/` contains:**

- `playground/` namespace for learning applications
- References external GitHub repos via raw URLs

**Example:**

```yaml
# apps/playground/kustomization.yaml
resources:
  - https://raw.githubusercontent.com/dxas90/learn-python/refs/heads/main/k8s/ks.yaml
```

## Secret Management

**SOPS with age:**

```bash
sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml
sops secret.sops.yaml  # Edit
sops --decrypt secret.sops.yaml  # View
```

**SOPS age key stored as Kubernetes secret:** `sops-age` in `flux-system` namespace

## Conventions

- **YAML schemas:** `# yaml-language-server: $schema=...` for validation
- **Namespace targeting:** Use `targetNamespace` in FluxCD Kustomizations
- **Labels:** Applied via `labels.yaml` LabelTransformer
- **Cluster overrides:** Place in `clusters/<type>/apps/`
- **Repository:** GitRepository `flux-system` is standard
- **Renovate:** Automated dependency updates via `renovate.json5`

## K3D Cluster Configuration

**`clusters/k3d/apps/`:**

- **MetalLB**: Load balancer (from `github.com/metallb/metallb/config/native?ref=v0.15.3`)
- **local-path**: Local storage provisioner

MetalLB uses two-stage deployment with `dependsOn` for proper initialization.

## Environment

**mise.toml:**

- Auto-set: `PROJECT_NAME`, `DEFAULT_IFACE`, `CURRENT_IP`, `CURRENT_EXTERNAL_IP`, `SECRET_KEY`
- Python venv: `/tmp/.venv-{{ PROJECT_NAME }}` (auto-created)
- Tools: All managed via mise - check availability with `mise list`
