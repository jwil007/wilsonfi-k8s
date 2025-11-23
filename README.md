# wilsonfi-k8s

GitOps repository for managing the wilsonfi.dev Kubernetes cluster using FluxCD.

## Cluster Info

- **Platform**: Talos Linux
- **Domain**: wilsonfi.dev
- **GitOps**: FluxCD
- **Secret Management**: SOPS with age encryption

## Repository Structure

```
.
├── apps/
│   ├── base/           # Base configurations for applications
│   └── prod/           # Production application deployments
│       └── <app>/      # Each app in its own folder with deployment, secrets, and monitoring
├── clusters/
│   └── wilsonfi-prod/  # Cluster-specific Flux configuration
├── infra/
│   ├── configs/        # Infrastructure configurations (ingress routes, issuers, etc.)
│   ├── controllers/    # Infrastructure controllers (cert-manager, traefik, etc.)
│   └── secrets/        # Encrypted infrastructure secrets
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

## VictoriaMetrics Monitoring

### Accessing Grafana

- URL: http://grafana.wilsonfi.dev
- Username: `admin`
- Get password: `kubectl get secret -n monitoring victoriametrics-k8s-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode`

### Adding Custom Scrape Targets

Create a `VMServiceScrape` in your app folder:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: myapp-metrics
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

VMAgent will automatically discover it (selectors are set to `{}`).

## Deploying New Applications

### Best Practice Structure

Each application should have its own folder under `apps/prod/<appname>/`:

```
apps/prod/myapp/
├── kustomization.yaml          # Lists all resources
├── myapp-deployment.yaml       # Namespace, Deployment, Service
├── myapp-secret.yaml           # SOPS-encrypted secrets
└── vmscrape.yaml               # Optional: metrics scraping config
```

### Example Deployment Workflow

```bash
# 1. Create app folder
mkdir -p apps/prod/myapp

# 2. Create deployment YAML with namespace, deployment, service

# 3. Create and encrypt secrets
sops -e -i apps/prod/myapp/myapp-secret.yaml

# 4. Create kustomization.yaml listing all resources

# 5. Add app folder to apps/prod/kustomization.yaml

# 6. Commit and push
git add apps/prod/myapp/
git commit -m "Add myapp deployment"
git push

# 7. Force reconcile
flux reconcile kustomization apps --with-source
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
