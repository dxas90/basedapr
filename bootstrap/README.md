# FluxCD Bootstrap Configuration

This directory contains the initial bootstrap configuration required to install FluxCD on a Kubernetes cluster.

## 📁 Contents

```
bootstrap/
├── kustomization.yaml    # Main bootstrap kustomization
├── README.md            # This file
└── [flux-components]    # FluxCD component definitions
```

## 🚀 Quick Start

### Prerequisites

Before bootstrapping, ensure you have:
- A running Kubernetes cluster
- `kubectl` installed and configured to access your cluster
- `flux` CLI installed ([installation guide](https://fluxcd.io/docs/installation/))
- Git repository access (SSH key pair recommended)
- `age` key for SOPS encryption (optional, for secret management)

### Bootstrap FluxCD

Apply the bootstrap configuration to install FluxCD:

```bash
kubectl apply --kustomize bootstrap
```

This command will:
- Install FluxCD Custom Resource Definitions (CRDs)
- Deploy FluxCD controllers in the `flux-system` namespace
- Set up the initial GitOps configuration

### Verify Installation

Check that FluxCD components are running:

```bash
# Check FluxCD pods
kubectl get pods -n flux-system

# Check FluxCD status
flux check

# View FluxCD components
flux get all
```

## 🔧 Configuration

### Git Repository Authentication

After bootstrap, configure Git repository access:

```bash
kubectl create secret generic flux-system \
    --namespace=flux-system \
    --from-file=identity=~/.ssh/id_rsa \
    --from-file=identity.pub=~/.ssh/id_rsa.pub \
    --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null)"
```

### SOPS Integration (Optional)

For secret encryption, create an age key secret:

```bash
kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=/path/to/age.key
```

## 🔄 Next Steps

After successful bootstrap:

1. **Deploy Cluster Configuration**:
   ```bash
   kubectl apply --kustomize clusters/<your-cluster-type>
   ```

2. **Monitor Deployment**:
   ```bash
   flux get kustomizations
   flux logs --follow
   ```

3. **Access Applications**:
   Wait for applications to deploy and configure ingress/load balancers as needed.

## 🛠️ Customization

### Custom FluxCD Configuration

To customize FluxCD components, modify the files in this directory before applying:

- Adjust resource limits/requests
- Configure additional controllers
- Modify reconciliation intervals
- Add custom annotations/labels

### Multi-Cluster Setup

For multiple clusters, you can:
- Use the same bootstrap configuration across clusters
- Customize per-cluster in the `clusters/` directory
- Implement cluster-specific GitOps repositories

## 🔍 Troubleshooting

### Common Issues

**Bootstrap fails with RBAC errors**:
```bash
# Check cluster admin permissions
kubectl auth can-i '*' '*' --all-namespaces
```

**FluxCD controllers not starting**:
```bash
# Check events in flux-system namespace
kubectl get events -n flux-system --sort-by='.lastTimestamp'

# Check controller logs
kubectl logs -n flux-system deployment/source-controller
```

**Git repository connection issues**:
```bash
# Verify secret creation
kubectl get secret flux-system -n flux-system

# Test Git connectivity from a pod
kubectl run test-git --rm -i --tty --image=alpine/git -- sh
```

### Useful Commands

```bash
# Force reconciliation
flux reconcile source git flux-system

# Suspend FluxCD
flux suspend kustomization flux-system

# Resume FluxCD
flux resume kustomization flux-system

# Check FluxCD version
flux version
```

## 📚 References

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [FluxCD Bootstrap Guide](https://fluxcd.io/docs/guides/repository-structure/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

> 💡 **Tip**: Keep this bootstrap configuration minimal and stable. Application-specific configurations should go in the `common/` or cluster-specific directories.
