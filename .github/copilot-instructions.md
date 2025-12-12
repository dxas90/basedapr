# GitOps FluxCD Kubernetes Infrastructure

This repository implements GitOps-based Kubernetes cluster management using FluxCD, Kustomize, Helm, and SOPS encryption.

## Architecture Overview

**Four-Layer Structure:**
1. **`bootstrap/`** - FluxCD installation (references `github.com/dxas90/common/install?ref=v0.4.0`)
2. **`apps/`** - Application deployments (references external GitHub repos, includes playground namespace)
3. **`clusters/<type>/`** - Cluster-specific config (only k3d is fully implemented; aws, azure, gcp, k0s, kind, metal are placeholders)
4. **`common/`** - Shared base infrastructure (currently just `default/echo` example app)

**GitOps Flow:** Git commits → FluxCD detects changes → Kustomizations reconcile → HelmReleases deploy apps

**Current State:** This is a minimal working example with one application (`echo`) and k3d cluster support. Most infrastructure mentioned in README.md (monitoring, observability, networking, dbms) is not yet implemented.

## Critical Workflows

### Bootstrap New Cluster (k3d only - other clusters not yet configured)
```bash
export CLUSTER=k3d
kubectl apply --kustomize bootstrap
cat ~/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin
kubectl create secret generic flux-system --namespace=flux-system \
  --from-file=identity=$HOME/.ssh/id_rsa --from-file=identity.pub=$HOME/.ssh/id_rsa.pub \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null | grep -v '^#')"
# Note: cluster-secrets.sops.yaml and cluster-settings.yaml don't exist yet for k3d
kubectl apply -k clusters/${CLUSTER}
```

### Alternative Bootstrap (from README with custom Git host)
```bash
kubectl apply --kustomize bootstrap
cat $HOME/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin
kubectl create secret generic flux-system \
  --namespace=flux-system \
  --from-file=identity=$HOME/Vaults/Safe/flux/fleet-flux-github \
  --from-file=identity.pub=$HOME/Vaults/Safe/flux/fleet-flux-github.pub \
  --from-literal=known_hosts="$(ssh-keyscan -p 4022 gitea.example.com 2>/dev/null | grep -v '^#')"
sops --decrypt clusters/${CLUSTER}/vars/cluster-secrets.sops.yaml | kubectl apply -f -
envsubst < clusters/${CLUSTER}/vars/cluster-settings.yaml | kubectl apply -f -
kubectl apply -k clusters/${CLUSTER}
```

### Check Deployment Status
```bash
flux get kustomizations
flux get helmreleases
flux logs --follow
```

### Tool Management
All tools (flux2, kubectl, k3d, sops, age, dapr, etc.) are managed via `mise` - see `mise.toml`. Run `mise use <tool>` to install.

## Application Structure Pattern

**Every app follows this exact structure:**

```
app-name/
├── app/
│   ├── kustomization.yaml          # Lists resources (helmrelease.yaml, ocirepository.yaml, etc.)
│   ├── ocirepository.yaml          # OCI chart source (or helmrepository.yaml for HTTP charts)
│   ├── helmrelease.yaml            # HelmRelease with chartRef pointing to OCIRepository
│   └── [configmaps/secrets].yaml   # Additional resources
└── install.yaml                    # FluxCD Kustomization pointing to ./common/app-name/app
```

**Example from `common/default/echo/`:**
- `install.yaml`: FluxCD Kustomization with `path: ./common/default/echo/app`, `sourceRef` points to GitRepository named `logging`
- `app/ocirepository.yaml`: Defines OCI chart source (`oci://ghcr.io/bjw-s-labs/helm/app-template`)
- `app/helmrelease.yaml`: HelmRelease with `chartRef.kind: OCIRepository` (NOT `chart.spec`)
- `app/kustomization.yaml`: Lists `helmrelease.yaml` and `ocirepository.yaml` as resources

**Adding New Apps:**
1. Create directory: `common/<category>/<app-name>/app/`
2. Add `ocirepository.yaml` or `helmrepository.yaml` in `app/`
3. Add `helmrelease.yaml` with `chartRef` (for OCI) or `chart.spec` (for HTTP repos)
4. Create `app/kustomization.yaml` listing all resources
5. Create `install.yaml` FluxCD Kustomization pointing to `./common/<category>/<app-name>/app`
6. Reference in parent: Add `- <app-name>/install.yaml` to `common/<category>/kustomization.yaml`

## FluxCD Resource Conventions

**HelmRelease with OCI charts:**
```yaml
spec:
  chartRef:
    kind: OCIRepository
    name: chart-name
```

**HelmRelease with HTTP Helm repos:**
```yaml
spec:
  chart:
    spec:
      chart: chart-name
      sourceRef:
        kind: HelmRepository
```

**FluxCD Kustomization (install.yaml):**
- `sourceRef.name: logging` (the GitRepository name for this repo - only used in `common/default/echo/install.yaml`)
- `sourceRef.name: flux-system` (used in cluster-specific apps like MetalLB)
- `path`: Relative path from repo root (e.g., `./common/default/echo/app`)
- `targetNamespace`: Where resources deploy
- `prune: true`, `wait: false` are common defaults

**Dependencies:** Use `dependsOn` in FluxCD Kustomizations (see `clusters/k3d/apps/metallb/ks.yaml` for example)

## Apps Layer Structure

**`apps/` directory contains external application references:**
- References `github.com/dxas90/common//common?ref=v0.4.0` for shared infrastructure
- `playground/` namespace: References raw GitHub URLs for multiple learning apps (learn-go, learn-java, learn-node, learn-php, learn-python, learn-ruby, learn-rust)
- `all.yaml` (currently unused): Contains GitRepository + Kustomization pairs for playground apps

**Pattern for external apps:**
```yaml
# apps/playground/kustomization.yaml
resources:
  - https://raw.githubusercontent.com/dxas90/learn-python/refs/heads/main/k8s/ks.yaml
```

## Secret Management

**SOPS with age:** All secrets encrypted with age keys
```bash
sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml
sops secret.sops.yaml  # Edit
sops --decrypt secret.sops.yaml  # View
```

**Cluster variables:**
- `clusters/<type>/vars/cluster-settings.yaml` - Plain config (used with `envsubst`)
- `clusters/<type>/vars/cluster-secrets.sops.yaml` - Encrypted secrets
- Reference with `postBuild.substituteFrom` in Kustomizations

## Key Conventions

- **YAML Language Server:** Most files use `# yaml-language-server: $schema=...` comments for validation
- **Namespace targeting:** `targetNamespace` in FluxCD Kustomizations, NOT namespace declarations in resources
- **Labels:** `common/labels.yaml` applies `app.kubernetes.io/usage: common` via LabelTransformer
- **Cluster-specific overrides:** Place in `clusters/<type>/apps/` (k3d has MetalLB and local-path storage)
- **Repository naming:**
  - GitRepository `logging` is used in `common/default/echo/install.yaml`
  - GitRepository `flux-system` is used in cluster-specific apps (MetalLB)
- **Renovate automation:** Configured in `renovate.json5` for automatic dependency updates

## K3D-Specific Configuration

**`clusters/k3d/apps/` contains local development infrastructure:**
- **MetalLB**: Load balancer for bare metal (references `github.com/metallb/metallb/config/native?ref=v0.15.3`)
  - Uses variable substitution: `${START_RANGE}`, `${END_RANGE}`, `${METALLB_MEMBERLIST_SECRET}`
  - Two-stage deployment: base resources + settings (with `dependsOn`)
- **local-path**: Storage class configuration in `kube-system` namespace
- References external infrastructure via `github.com/dxas90/common//common?ref=v0.4.0` in apps layer

## Environment Context

**mise.toml environment variables:**
- `PROJECT_NAME`, `DEFAULT_IFACE`, `CURRENT_IP`, `CURRENT_EXTERNAL_IP` auto-set from commands
- Python venv auto-activates at `/tmp/.venv-{{ PROJECT_NAME }}`

When creating commands or scripts, use `mise` for tool availability checks rather than assuming tools are globally installed.
