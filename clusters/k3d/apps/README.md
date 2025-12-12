# K3D Cluster-Specific Applications

This directory contains applications and configurations specific to the k3d local development cluster.

## 📁 Directory Structure

```
k3d/apps/
├── kustomization.yaml      # Main apps kustomization
├── labels.yaml            # Common labels for k3d apps
├── README.md              # This file
├── local-path/            # Local path storage configuration
└── metallb/               # MetalLB load balancer for local development
```

## 🏗️ K3D-Specific Applications

### Local Path Storage
- **Purpose**: Provides local storage for development workloads
- **Type**: Built-in k3d storage solution
- **Use Case**: Development testing of stateful applications

### MetalLB
- **Purpose**: Load balancer implementation for bare metal Kubernetes
- **Type**: Layer 2/BGP load balancer
- **Use Case**: Exposes services with LoadBalancer type in local k3d environment

## 🚀 Usage

These applications are automatically deployed when applying the k3d cluster configuration:

```bash
kubectl apply --kustomize clusters/k3d
```

## 🔧 Local Development Features

### Storage
- Persistent volumes backed by local host storage
- Suitable for development and testing
- Data persists across pod restarts but not cluster destruction

### Load Balancing
- MetalLB provides LoadBalancer service support
- Allocates IP addresses from a configurable pool
- Enables testing of ingress controllers and external access

### Resource Limits
- Optimized for local development workloads
- Lower resource requests/limits compared to production
- Fast iteration and testing capabilities

## 🛠️ Customization

### Storage Configuration
Modify `local-path/` configurations to:
- Change storage class parameters
- Adjust volume provisioning settings
- Configure retention policies

### Load Balancer Pool
Update MetalLB configuration to:
- Change IP address pool ranges
- Configure L2/BGP announcements
- Adjust load balancer behavior

## 📊 Monitoring

K3D cluster includes basic monitoring capabilities:
- Resource usage tracking
- Application health checks
- Development-focused metrics

## 🔍 Troubleshooting

### Storage Issues
```bash
# Check storage class
kubectl get storageclass

# Check persistent volumes
kubectl get pv

# Check volume mounts
kubectl describe pod <pod-name>
```

### LoadBalancer Issues
```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check service LoadBalancer status
kubectl get svc -o wide

# Check MetalLB configuration
kubectl get configmap -n metallb-system
```

## 📚 References

- [k3d Documentation](https://k3d.io/)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [Kubernetes Local Development](https://kubernetes.io/docs/setup/learning-environment/)

---

> 💡 **Note**: These configurations are optimized for local development. Production clusters should use the configurations in their respective cluster directories.
