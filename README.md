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

### 1. Install Dependencies

```bash
cd /path/to/monitoringHelmTemplate
helm dependency update
```

This downloads the required webdevops charts (azure-keyvault-exporter, azure-resourcemanager-exporter).

### 2. Configure Values

Create a custom values file or edit `values.yaml`:

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

### 3. Deploy

```bash
helm install monitoring-stack . \
  --namespace monitoring \
  --create-namespace \
  --wait
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
# Update dependencies if Chart.yaml changed
helm dependency update

# Upgrade release
helm upgrade monitoring-stack . \
  --namespace monitoring \
  --values values-production.yaml
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

- **[HELM_VALUES.md](./HELM_VALUES.md)** - Complete values reference with all parameters
- **[DEPLOYMENT.md](./DEPLOYMENT.md)** - Detailed deployment procedures and checklists
- **[VERIFICATION.md](./VERIFICATION.md)** - Testing and validation procedures

## 🗂️ Chart Structure

```
monitoringHelmTemplate/
├── Chart.yaml                      # Chart metadata + dependencies
├── values.yaml                     # Default configuration
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
└── charts/                         # Dependencies (after helm dep update)
```

## 🔧 Environment-Specific Deployments

### Development

```bash
helm install monitoring-stack . \
  -n monitoring-dev \
  --create-namespace \
  -f values.yaml \
  -f values-dev.yaml
```

### Production

```bash
helm install monitoring-stack . \
  -n monitoring \
  --create-namespace \
  -f values.yaml \
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

## 🤝 Contributing

When making changes:

1. Update `values.yaml` with new options
2. Update `HELM_VALUES.md` documentation
3. Test with `helm lint` and `helm template`
4. Increment chart version in `Chart.yaml`

## 📝 Migration from Kustomize

This chart was converted from a Kustomize-based deployment. The original manifests are preserved in `manifests/` for reference.

To migrate:

1. Backup existing deployment: `kubectl get all -n monitoring -o yaml > backup.yaml`
2. Uninstall Kustomize deployment: `kubectl delete -k .`
3. Follow Quick Start guide above
4. Verify all metrics are being collected

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
