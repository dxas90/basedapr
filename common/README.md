# Common Infrastructure Applications

This directory contains shared infrastructure applications that are deployed across all cluster environments. Each application follows a standardized structure for consistent deployment and management.

## 📁 Directory Structure

Each application is organized using the following pattern:

```
app-name/
├── app/
│   ├── kustomization.yaml          # Application resources
│   ├── helmrepository.yaml         # Helm repository (if using Helm)
│   ├── ocirepository.yaml          # OCI repository (if using OCI charts)
│   ├── gitrepository.yaml          # Git repository (if using Git sources)
│   ├── helmrelease.yaml            # Helm release configuration
│   └── [additional-resources].yaml # ConfigMaps, Secrets, etc.
├── install.yaml                    # FluxCD Kustomization for deployment
├── ks.yaml                         # Legacy FluxCD Kustomization (being phased out)
├── kustomization.yaml              # Base kustomization (if needed)
└── namespace.yaml                  # Namespace definition (if applicable)
```

## 🏗️ Application Categories

### Core Infrastructure
- **cert-manager/**: Automated TLS certificate management using Let's Encrypt and other ACME providers
- **external-secrets/**: Kubernetes External Secrets Operator for integrating with external secret management systems
- **vault/**: HashiCorp Vault for centralized secret management and encryption

### Kubernetes System (`kube-system/`)
- **cilium/**: Advanced networking, security, and observability with eBPF
- **coredns/**: DNS service for the cluster
- **csi-driver-nfs/**: NFS Container Storage Interface driver
- **metrics-server/**: Cluster-wide resource usage metrics
- **external-secrets/**: External secrets stores configuration

### Kubernetes Tools (`kube-tools/`)
- **kyverno/**: Policy engine for Kubernetes security and governance
- **node-feature-discovery/**: Detects hardware features available on each node

### Observability Platform (`observability/`)
- **alertmanager/**: Alert routing and notification management
- **alloy/**: OpenTelemetry-compatible telemetry collection
- **blackbox-exporter/**: External service monitoring and probing
- **fluent-bit/**: Log collection and forwarding
- **gatus/**: Service health dashboard and status page
- **goldilocks/**: Resource recommendation for right-sizing pods
- **grafana/**: Visualization platform for metrics, logs, and traces
- **kube-prometheus-stack/**: Complete monitoring solution (Prometheus, Grafana, AlertManager)
- **loki/**: Log aggregation system optimized for Kubernetes
- **node-exporter/**: Hardware and OS metrics collection
- **silence-operator/**: Automated alert silencing management
- **victoria-logs/**: High-performance log database
- **vpa/**: Vertical Pod Autoscaler for automatic resource scaling
- **alloy/**: Grafana Alloy for telemetry data collection
- **victoria-logs/**: High-performance log database
- **alertmanager/**: Alert routing and management
- **blackbox-exporter/**: Blackbox monitoring for endpoints
- **node-exporter/**: Hardware and OS metrics
- **silence-operator/**: Automated alert silencing

### Networking (`networking/`)
- **traefik/**: Modern reverse proxy and load balancer with automatic service discovery
- **gateway-api/**: Next-generation ingress APIs for Kubernetes

### Database Management Systems (`dbms/`)
- **cloudnative-pg/**: PostgreSQL operator for cloud-native deployments
- **cockroachdb/**: Distributed SQL database for global applications
- **nats/**: Cloud-native messaging system
- **dragonfly-operator/**: Redis-compatible in-memory data store with enhanced performance

## 🚀 Application Deployment

### Standard Deployment Pattern

All applications use the same deployment pattern through FluxCD:

1. **Repository Definition**: Each app includes its own repository configuration
   - `helmrepository.yaml` for Helm charts
   - `ocirepository.yaml` for OCI-based charts
   - `gitrepository.yaml` for Git-based configurations

2. **Application Resources**: Defined in the `app/` subdirectory
   - Helm releases, configurations, secrets, etc.

3. **FluxCD Integration**: `install.yaml` defines the FluxCD Kustomization
   - Points to the application's `app/` directory
   - Manages dependencies and deployment order
   - Handles namespace targeting and reconciliation

### Adding a New Application

1. **Create Directory Structure**:
   ```bash
   mkdir -p common/new-app/app
   ```

2. **Add Repository Definition** (choose appropriate type):
   ```yaml
   # app/helmrepository.yaml
   apiVersion: source.toolkit.fluxcd.io/v1
   kind: HelmRepository
   metadata:
     name: new-app
     namespace: flux-system
   spec:
     interval: 1h
     url: https://charts.example.com/
   ```

3. **Create Application Configuration**:
   ```yaml
   # app/helmrelease.yaml
   apiVersion: helm.toolkit.fluxcd.io/v2
   kind: HelmRelease
   metadata:
     name: new-app
     namespace: new-app
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

4. **Create App Kustomization**:
   ```yaml
   # app/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   namespace: new-app
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
   Add to the appropriate category's `kustomization.yaml`:
   ```yaml
   resources:
     - new-app/install.yaml
   ```

## 🔧 Configuration Management

### Environment-Specific Overrides

Applications can be customized per cluster using Kustomize overlays in the cluster-specific directories:

```yaml
# clusters/production/kustomization.yaml
patchesStrategicMerge:
  - production-overrides.yaml
```

### Secret Management

- **External Secrets**: Integrated with various secret backends
- **SOPS**: Git-native secret encryption for sensitive configurations
- **Vault**: Centralized secret management and dynamic secrets

### Dependencies

Application dependencies are managed through FluxCD:

```yaml
# install.yaml
spec:
  dependsOn:
    - name: cert-manager
      namespace: cert-manager
    - name: external-secrets
      namespace: external-secrets
```

## 📊 Monitoring and Health

### Application Health Checks

- All applications include health checks and monitoring configurations
- Prometheus metrics collection enabled by default
- Grafana dashboards provided where applicable
- Alert rules configured for critical components

### Status Monitoring

Use FluxCD commands to monitor application status:

```bash
# Check all applications
flux get kustomizations

# Check specific application
flux get helmrelease new-app

# Force reconciliation
flux reconcile kustomization new-app
```

## 🛠️ Maintenance

### Updates and Upgrades

1. **Automated Updates**: Renovate bot manages dependency updates
2. **Manual Updates**: Update chart versions in `helmrelease.yaml`
3. **Rollbacks**: Use FluxCD suspend/resume for quick rollbacks

### Troubleshooting

Common troubleshooting steps:

```bash
# Check application status
kubectl describe helmrelease new-app -n new-app

# Check FluxCD logs
flux logs --follow

# Check specific controller logs
kubectl logs -n flux-system deployment/helm-controller
```

## 📚 Best Practices

1. **Namespace Isolation**: Each application should have its own namespace
2. **Resource Limits**: Always define resource requests and limits
3. **Security Policies**: Follow security best practices and policy compliance
4. **Documentation**: Maintain clear documentation for custom configurations
5. **Testing**: Test changes in development clusters before production
6. **Monitoring**: Ensure proper monitoring and alerting for all applications

---

For more information about specific applications, refer to their individual directories and the main project documentation.
