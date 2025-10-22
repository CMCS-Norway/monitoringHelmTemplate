# Monitoring Stack Helm Chart

A production-ready Helm chart for deploying a complete monitoring stack with Azure and network exporters.

## 🎯 What This Chart Deploys

This chart manages four monitoring exporters with unified configuration:

- **Azure KeyVault Exporter** - Monitors Azure Key Vaults (certificate expiry, secret access)
- **Azure Resource Manager Exporter** - Monitors Azure resources (costs, health, security, reservations)
- **Fortigate Exporter** - Monitors Fortigate firewalls (25+ locations, VPN status, traffic, sessions)
- **Blackbox Exporter** - HTTP/HTTPS/TCP/ICMP probing for endpoint monitoring

## 📋 Prerequisites

- Kubernetes 1.19+
- Helm 3.8+
- [External Secrets Operator](https://external-secrets.io/) installed
- ClusterSecretStore configured (default: `azure-backend`)
- Prometheus Operator (optional, for ServiceMonitors)

## 🚀 Quick Start

### 1. Add Helm Repository

```bash
# Add the CMCS monitoring Helm repository
helm repo add cmcs-monitoring https://cmcs-norway.github.io/monitoringHelmTemplate/

# Update repository index
helm repo update

# Search for available charts
helm search repo cmcs-monitoring
```

### 2. Install the Chart

```bash
# Install with default values
helm install monitoring-stack cmcs-monitoring/monitoring-stack \
  --namespace monitoring \
  --create-namespace

# Or install with custom values
helm install monitoring-stack cmcs-monitoring/monitoring-stack \
  --namespace monitoring \
  --create-namespace \
  -f custom-values.yaml
```

### 3. Configure Values

Create a custom values file (`custom-values.yaml`):

```yaml
global:
  namespace: monitoring
  customer: "your-company"
  customerLabel: "Your Company Name"

azureKeyVaultExporter:
  enabled: true
  env:
    AZURE_SUBSCRIPTION_ID: "your-azure-subscription-id"
    KEYVAULT_FILTER: "where tags['monitoring'] == 'true'"

azureResourceManagerExporter:
  enabled: true

fortigateExporter:
  enabled: true
  scrapeConfig:
    targets:
      - https://fw-location1.fortidyndns.com
      - https://fw-location2.fortidyndns.com:10443

blackboxExporter:
  enabled: true
  config:
    modules:
      http_2xx:
        timeout: 10s
        http:
          valid_status_codes: [200, 301, 302]
```

### 4. Verify

```bash
# Check pods are running
kubectl get pods -n monitoring

# Check ExternalSecrets synced
kubectl get externalsecrets -n monitoring

# View deployment notes
helm status monitoring-stack -n monitoring
```

## 🏗️ Architecture

### Hybrid Chart Structure

- **Dependency Charts** (managed by Helm)
  - `azure-keyvault-exporter` v1.0.12 (webdevops)
  - `azure-resourcemanager-exporter` v1.3.5 (webdevops)
  - `prometheus-blackbox-exporter` v11.3.1 (prometheus-community)

- **Custom Deployments** (managed by templates)
  - `fortigate-exporter` v3.0.0 (custom image)

### Resources Created

| Component | Resources |
|-----------|-----------|
| **Azure KeyVault** | Deployment, Service, ServiceMonitor, ExternalSecret |
| **Azure Resource Manager** | Deployment, Service, ConfigMap, ScrapeConfig |
| **Fortigate** | Deployment, Service, ExternalSecret, ScrapeConfig |
| **Blackbox** | Deployment, Service, ConfigMap |
| **Global** | ExternalSecret (azure-config) |

**Total:** ~18-20 Kubernetes resources when fully enabled

## ⚙️ Configuration

### Enable/Disable Components

Each exporter can be independently controlled:

```yaml
azureKeyVaultExporter:
  enabled: true  # Set to false to disable

azureResourceManagerExporter:
  enabled: true  # Set to false to disable

fortigateExporter:
  enabled: true  # Set to false to disable

blackboxExporter:
  enabled: true  # Set to false to disable
```

### Azure KeyVault Exporter

```yaml
azureKeyVaultExporter:
  enabled: true
  replicas: 1
  
  env:
    AZURE_SUBSCRIPTION_ID: "sub-id-1 sub-id-2"  # Space-separated
    KEYVAULT_FILTER: "where tags['monitoring'] == 'true'"
  
  resources:
    limits:
      cpu: 500m
      memory: 200Mi
    requests:
      cpu: 100m
      memory: 200Mi
  
  prometheus:
    monitor:
      enabled: true  # Creates ServiceMonitor
```

### Azure Resource Manager Exporter

```yaml
azureResourceManagerExporter:
  enabled: true
  
  additionalResources:
    configMap:
      enabled: true
      data:
        collectors:
          general:
            scrapeTime: 24h
          costs:
            scrapeTime: 24h
          defender:
            scrapeTime: 24h
          resourceHealth:
            scrapeTime: 24h
```

### Blackbox Exporter

```yaml
blackboxExporter:
  enabled: true
  
  # Configure probe modules
  config:
    modules:
      http_2xx:
        timeout: 10s
        http:
          valid_status_codes: [200, 301, 302, 307, 401, 403]
      
      # Add custom modules as needed
      http_post_2xx:
        prober: http
        timeout: 5s
        http:
          method: POST
          valid_status_codes: [200, 201]
  
  resources:
    requests:
      cpu: 5m
      memory: 30Mi
    limits:
      cpu: 10m
      memory: 50Mi
  
  # Enable ScrapeConfig for endpoint monitoring
  scrapeConfig:
    enabled: true
    targets:
      - https://example.com
      - https://api.example.com/health
      - http://internal-service.svc.cluster.local:8080
    params:
      module: [http_2xx]  # Use http_2xx probe module
```

**Common use cases:**
- HTTP/HTTPS endpoint monitoring
- SSL certificate expiration checks
- TCP port availability
- ICMP ping checks
- DNS resolution validation

**Example: Multi-module probing**
```yaml
# Create multiple ScrapeConfigs for different probe types
blackboxExporter:
  config:
    modules:
      http_2xx:
        http:
          valid_status_codes: [200]
      http_post:
        http:
          method: POST
      tcp_connect:
        prober: tcp
      icmp:
        prober: icmp
```

### Fortigate Exporter

```yaml
fortigateExporter:
  enabled: true
  
  image:
    registry: ghcr.io
    repository: cmcs-norway/fortigate-exporter-image
    tag: "v3.0.0"
  
  replicas: 1
  
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
  
  scrapeConfig:
    enabled: true
    scrapeInterval: 5m
    targets:
      - https://fw-site1.fortidyndns.com
      - https://fw-site2.fortidyndns.com:10443
      # Add more firewalls as needed
```

### External Secrets Configuration

All credentials are managed via External Secrets Operator:

```yaml
global:
  azureSecrets:
    enabled: true
    externalSecret:
      name: azure-config
      refreshInterval: 12h
      secretStoreRef:
        kind: ClusterSecretStore
        name: azure-backend
      remoteKeys:
        clientId: azure-client-id
        clientSecret: azure-client-secret
        tenantId: azure-tenant-id
```

## 📊 Prometheus Integration

### ServiceMonitors

Azure KeyVault Exporter includes a ServiceMonitor (Prometheus Operator):

```yaml
azureKeyVaultExporter:
  prometheus:
    monitor:
      enabled: true
      relabelings:
        - action: replace
          targetLabel: customer
          replacement: "Your Company"
```

### ScrapeConfigs

The chart creates ScrapeConfigs for Prometheus Agent mode:

- Fortigate firewalls (probe-based scraping)
- Azure Resource Manager (direct scraping)
- Azure Resource Graph (optional)

## 🔐 Security

### Credentials Management

- **No secrets in Git** - All credentials stored in Azure Key Vault
- **ExternalSecrets** - Auto-sync every 12 hours
- **Read-only volumes** - Secrets mounted as read-only

### Pod Security

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  fsGroup: 1000

containerSecurityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

## 🔄 Management

### Upgrade

```bash
# Update Helm repository
helm repo update

# Upgrade to latest version
helm upgrade monitoring-stack cmcs-monitoring/monitoring-stack \
  --namespace monitoring \
  --values custom-values.yaml
```

### Rollback

```bash
# View history
helm history monitoring-stack -n monitoring

# Rollback to previous version
helm rollback monitoring-stack -n monitoring
```

### Uninstall

```bash
# Remove all resources
helm uninstall monitoring-stack -n monitoring

# Optional: Clean up ExternalSecrets
kubectl delete externalsecrets --all -n monitoring
```

## 🧪 Testing

### Pre-Deployment Validation

```bash
# Lint the chart
helm lint .

# Render templates to verify
helm template monitoring-stack . --debug | less

# Dry-run installation
helm install monitoring-stack . --dry-run --debug
```

### Post-Deployment Verification

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Verify ExternalSecrets synced
kubectl get externalsecrets -n monitoring
kubectl describe externalsecret azure-config -n monitoring

# Test metrics endpoints
kubectl port-forward -n monitoring svc/azure-kv-exporter 8080:8080
curl http://localhost:8080/metrics
```

## 🐛 Troubleshooting

### Pods Not Starting

```bash
# Check pod logs
kubectl logs -n monitoring <pod-name>

# Common issues:
# - ExternalSecret not synced
# - Missing ClusterSecretStore
# - Invalid credentials in Azure Key Vault
```

### ExternalSecret Not Syncing

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n monitoring
kubectl describe externalsecret azure-config -n monitoring

# Verify ClusterSecretStore exists
kubectl get clustersecretstore azure-backend

# Check External Secrets Operator is running
kubectl get pods -n external-secrets-system
```

### Dependencies Missing

```bash
# Error: "chart directory is missing these dependencies"
# Solution:
helm dependency update
```

### Metrics Not Appearing in Prometheus

```bash
# Check ServiceMonitors created
kubectl get servicemonitors -n monitoring

# Check ScrapeConfigs created
kubectl get scrapeconfigs -n monitoring

# Verify Prometheus can reach services
kubectl get svc -n monitoring
```

## 📚 Documentation

### Quick Links

- **[Quick Start](#-quick-start)** - Get started in 3 steps
- **[Configuration Examples](#️-configuration)** - Common use cases
- **[Troubleshooting](#-troubleshooting)** - Common issues and solutions

### Detailed Guides

- **[HELM_VALUES.md](./docs/HELM_VALUES.md)** - Complete values reference with all parameters
- **[DEPLOYMENT.md](./docs/DEPLOYMENT.md)** - Detailed deployment procedures and checklists
- **[VERIFICATION.md](./docs/VERIFICATION.md)** - Testing and validation procedures
- **[RELEASE_GUIDE.md](./docs/RELEASE_GUIDE.md)** - How to create releases (automated versioning)

## 🗂️ Repository Structure

```
monitoringHelmTemplate/
├── Chart.yaml                           # Chart metadata + dependencies
├── values.yaml                          # Default configuration
├── README.md                            # This file
├── renovate.json                        # Renovate config for dependency updates
├── docs/                                # Documentation
│   ├── DEPLOYMENT.md                    # Deployment guide
│   ├── HELM_VALUES.md                   # Complete values reference
│   ├── VERIFICATION.md                  # Testing and validation
│   └── RELEASE_GUIDE.md                 # Release process
├── templates/                           # Kubernetes resource templates
│   ├── _helpers.tpl                     # Template helper functions
│   ├── NOTES.txt                        # Post-install notes
│   ├── secrets.yaml                     # Global ExternalSecret
│   ├── kv-exporter-secret.yaml         # KeyVault ExternalSecret
│   ├── kv-exporter-scrapeconfig.yaml   # Optional ScrapeConfig
│   ├── rm-exporter-configmap.yaml      # Resource Manager ConfigMap
│   ├── fortigate-deployment.yaml       # Fortigate Deployment
│   ├── fortigate-service.yaml          # Fortigate Service
│   ├── fortigate-secret.yaml           # Fortigate ExternalSecret
│   ├── fortigate-scrapeconfig.yaml     # Fortigate ScrapeConfig
│   ├── blackbox-scrapeconfig.yaml      # Blackbox ScrapeConfig
│   └── scrapeconfigs.yaml              # Additional ScrapeConfigs
├── .github/
│   └── workflows/
│       ├── release.yml                  # Release automation
│       └── renovate.yml                 # Dependency updates
└── charts/                              # Dependencies (after helm dep update)
```

## 🔧 Environment-Specific Deployments

### Development

```bash
# Add the repository
helm repo add cmcs-monitoring https://cmcs-norway.github.io/monitoringHelmTemplate/
helm repo update

# Install with dev values
helm install monitoring-stack cmcs-monitoring/monitoring-stack \
  -n monitoring-dev \
  --create-namespace \
  -f values-dev.yaml
```

### Production

```bash
# Add the repository
helm repo add cmcs-monitoring https://cmcs-norway.github.io/monitoringHelmTemplate/
helm repo update

# Install with production values
helm install monitoring-stack cmcs-monitoring/monitoring-stack \
  -n monitoring \
  --create-namespace \
  -f values-prod.yaml \
  --atomic \
  --timeout 10m
```

## 📈 Monitoring Capabilities

### Azure Metrics
- Key Vault certificate expiration
- Resource health across subscriptions
- Cost tracking (by subscription, resource group)
- Security recommendations (Defender)
- Reservation utilization

### Network Metrics
- Fortigate system status
- VPN connection counts
- Interface traffic statistics
- Active sessions
- CPU and memory utilization
- License expiration

### HTTP/Endpoint Metrics
- HTTP response codes
- Response times
- SSL certificate validity
- DNS resolution
- TCP connectivity

### Scrape Intervals
- **Azure KeyVault:** 1 minute
- **Azure Resources:** 24 hours (configurable per collector)
- **Fortigate:** 5 minutes
- **Blackbox:** On-demand (via Prometheus scrape configs)

## 🤖 Automated Dependency Updates

This chart uses **Renovate** to automatically monitor and update Helm chart dependencies.

### How It Works

- **Schedule:** Every Monday at 9 AM (Oslo time)
- **Monitors:** `Chart.yaml` dependencies (Azure exporters, Prometheus exporters)
- **Creates PRs:** Automatic pull requests with updates
- **Grouped:** Related dependencies updated together
- **Security:** Vulnerability alerts enabled

### Dependency Groups

1. **Azure Exporters**
   - `azure-keyvault-exporter`
   - `azure-resourcemanager-exporter`

2. **Prometheus Exporters**
   - `prometheus-blackbox-exporter`

Updates are reviewed and tested before being merged.

## 🤝 Contributing

### Making Changes

1. Create a feature branch
2. Make your changes
3. Update `values.yaml` with new options (if applicable)
4. Update documentation in `docs/` folder
5. Test with `helm lint` and `helm template`
6. Create a pull request

### Creating a Release

See **[RELEASE_GUIDE.md](./docs/RELEASE_GUIDE.md)** for detailed instructions.

Quick version:
```bash
# Just create and push a tag - version is automated!
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1
```

The workflow automatically:
- Updates Chart.yaml version from tag
- Packages the chart
- Creates GitHub Release
- Publishes to Helm repository

## 📦 Local Development

If you want to develop or test changes locally:

```bash
# Clone the repository
git clone https://github.com/CMCS-Norway/monitoringHelmTemplate.git
cd monitoringHelmTemplate

# Update dependencies
helm dependency update

# Test the chart
helm lint .
helm template monitoring-stack . --debug

# Install from local directory
helm install monitoring-stack . \
  --namespace monitoring \
  --create-namespace
```

## 📄 License

[Your License Here]

## 🆘 Support

For issues or questions:
- Review the documentation in this repository
- Check the [webdevops charts documentation](https://github.com/webdevops/helm-charts)
- Open an issue in your repository

---

**Chart Version:** 1.0.0  
**Last Updated:** 2025-10-22  
**Status:** Production Ready
