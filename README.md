# wilsonfi-k8s

Homelab k8s cluster. Using GitOps with fluxCD to manage configuration and deployment.

## Cluster Info

Three node cluster installed on bare-metal. Using HP G4 MP9 mini PCs purchased for a song from eBay. 32GB ram per node, i5 8500T.

Each node is running the control plane as well as worker jobs. This is a pragmatic design for a homelab, giving HA for the control plane, but not requiring seperate worker nodes.

After deciding which k8s distro to go with (k3s, kubeadm, Talos), I ended up using Talos. Unlike other distros, Talos is a full Linux image - it is not run on top of an existing OS. It is a specialized Linux distro that does ONE thing, which is run k8s. No shell, no SSH. Only an API interfaced with through talosctl. At first this was strange, but it's appealing to abstract away with Linux OS itself when using k8s. K8s is complicated enough on it's own, so it's nice to know I can't break something on the node's OS. 

- **K8s Distro**: Talos Linux
- **GitOps**: FluxCD
- **Secret Management**: SOPS with age encryption
- **Storage**: Longhorn (In cluster storage, dedicated NVME drives for use by Longhorn)
- **Monitoring Stack**: Victoria Metrics with Grafana (https://docs.victoriametrics.com/helm/victoria-metrics-k8s-stack/)


## Repository Structure

```
.
├── apps/
│   └── prod/           # Production application deployments
│       └── <app>/      # Each app in its own folder with deployment, secrets, and monitoring
├── clusters/
│   └── wilsonfi-prod/  # Cluster-specific Flux configuration (right now I only have a single cluster)
├── infra/
│   └── <component>/          # Each component of the infra has it's own dir, containing deployment yamls, secrets, ingress config, etc.
```

## Flux Reconcile Cheat Sheet

### Common Commands

```bash
# Reconcile GitRepository (fetch latest from Git)
flux reconcile source git flux-system

# Reconcile a specific Kustomization
flux reconcile kustomization infra-controllers
flux reconcile kustomization infra-configs
flux reconcile kustomization apps

# Reconcile Kustomization AND pull fresh Git source (most common after git push)
flux reconcile kustomization infra-controllers --with-source
flux reconcile kustomization apps --with-source

# Reconcile a HelmRelease
flux reconcile helmrelease victoriametrics-k8s-stack -n monitoring

# Reconcile HelmRepository (update chart index)
flux reconcile source helm victoriametrics -n flux-system
```

### Status Commands

```bash
# Get status of everything
flux get all
flux get kustomizations
flux get helmreleases -A
flux get sources all

# Check specific resources
flux get sources git
flux get sources helm
```

### Suspend/Resume

```bash
# Suspend reconciliation (useful for testing)
flux suspend kustomization apps
flux suspend helmrelease <name> -n <namespace>

# Resume reconciliation
flux resume kustomization apps
flux resume helmrelease <name> -n <namespace>
```

### Force Reconcile Everything

```bash
# Reconcile everything in order
flux reconcile source git flux-system && \
flux reconcile kustomization flux-system && \
flux reconcile kustomization infra-controllers && \
flux reconcile kustomization infra-configs && \
flux reconcile kustomization apps
```

## SOPS Secret Management

### Encrypting Secrets

```bash
# Encrypt a secret file in-place
sops -e -i path/to/secret.yaml

# Decrypt to view (doesn't modify file)
sops -d path/to/secret.yaml

# Edit encrypted secret
sops path/to/secret.yaml
```

### Creating New Secrets

```bash
# 1. Create plain YAML secret
cat > apps/prod/myapp/secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: myapp
type: Opaque
stringData:
  username: "myuser"
  password: "mypassword"
EOF

# 2. Encrypt it with SOPS
sops -e -i apps/prod/myapp/secret.yaml

# 3. Commit and push
git add apps/prod/myapp/secret.yaml
git commit -m "Add myapp secret"
git push
```

## Useful kubectl Commands

```bash
# Get all resources in a namespace
kubectl get all -n <namespace>

# Check pod logs
kubectl logs -n <namespace> <pod-name>
kubectl logs -n <namespace> deployment/<deployment-name>

# Describe resources for debugging
kubectl describe pod -n <namespace> <pod-name>
kubectl describe deployment -n <namespace> <deployment-name>

# Get events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Port forward for local access
kubectl port-forward -n <namespace> svc/<service-name> 8080:80
```

## Infrastructure Components

- **Cert-Manager**: TLS certificate management with Let's Encrypt
- **Traefik**: Ingress controller at 10.0.50.200
- **Longhorn**: Distributed storage
- **MetalLB**: LoadBalancer implementation
- **VictoriaMetrics**: Metrics collection and storage (12-month retention, 150Gi)
- **Grafana**: Metrics visualization
- **Flux Capacitor UI**: Handy UI to check Flux status

## Troubleshooting

### Flux not reconciling

```bash
# Check Flux system health
flux check

# Check for errors
flux get all
kubectl get pods -n flux-system

# View controller logs
kubectl logs -n flux-system deploy/kustomize-controller
kubectl logs -n flux-system deploy/source-controller
```

### SOPS decryption failing

```bash
# Verify age key is present in cluster
kubectl get secret -n flux-system sops-age

# Test decryption locally
sops -d path/to/secret.yaml
```

### HelmRelease stuck

```bash
# Check HelmRelease status
flux get helmreleases -A
kubectl describe helmrelease -n <namespace> <name>

# Force reconcile
flux reconcile helmrelease <name> -n <namespace>

# Nuclear option: suspend and resume
flux suspend helmrelease <name> -n <namespace>
flux resume helmrelease <name> -n <namespace>
```
