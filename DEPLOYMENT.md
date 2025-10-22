# Quick Deployment Guide

## Prerequisites Checklist

- [ ] Helm 3.8+ installed
- [ ] kubectl configured for target cluster
- [ ] External Secrets Operator installed
- [ ] ClusterSecretStore `azure-backend` configured
- [ ] Azure Service Principal credentials stored in secret backend

## Deployment Steps

### 1. Clone/Navigate to Chart Directory

```bash
cd /Users/marius/repo/cmcs/monitoringHelmTemplate
```

### 2. Update Helm Dependencies

```bash
helm dependency update
```

This downloads the webdevops charts into `charts/` directory.

### 3. Review Configuration

Edit `values.yaml` or create environment-specific values file:

```bash
# For production
cp values.yaml values-production.yaml
vi values-production.yaml
```

**Required customizations:**
- `global.namespace` - Target namespace name
- `global.customer` - Customer identifier  
- `azureKeyVaultExporter.env.AZURE_SUBSCRIPTION_ID` - Your Azure subscription(s)
- `azureResourceManagerExporter.additionalResources.configMap.data` - Collector settings

### 4. Validate Chart

```bash
# Lint the chart
helm lint .

# Dry-run to see what will be created
helm install monitoring-stack . \
  --namespace monitoring \
  --create-namespace \
  --dry-run \
  --debug
```

### 5. Install Chart

```bash
helm install monitoring-stack . \
  --namespace monitoring \
  --create-namespace \
  --values values-production.yaml \
  --wait
```

### 6. Verify Installation

```bash
# Check Helm release status
helm status monitoring-stack -n monitoring

# Check all pods are running
kubectl get pods -n monitoring

# Check ExternalSecrets are synced
kubectl get externalsecrets -n monitoring

# Verify secrets were created
kubectl get secrets -n monitoring

# Check ConfigMaps
kubectl get configmaps -n monitoring
```

### 7. Test Metrics Endpoint

```bash
# Port-forward to KeyVault Exporter
kubectl port-forward -n monitoring svc/azure-kv-exporter 8080:8080

# In another terminal, curl the metrics
curl http://localhost:8080/metrics
```

## Upgrade Existing Deployment

```bash
# Update dependencies if Chart.yaml changed
helm dependency update

# Upgrade release
helm upgrade monitoring-stack . \
  --namespace monitoring \
  --values values-production.yaml \
  --wait
```

## Rollback

```bash
# View release history
helm history monitoring-stack -n monitoring

# Rollback to previous version
helm rollback monitoring-stack -n monitoring

# Rollback to specific revision
helm rollback monitoring-stack 2 -n monitoring
```

## Uninstall

```bash
# Remove Helm release
helm uninstall monitoring-stack -n monitoring

# Clean up ExternalSecrets (optional)
kubectl delete externalsecrets --all -n monitoring

# Delete namespace (optional)
kubectl delete namespace monitoring
```

## Common Issues

### Issue: Dependencies not found

**Solution:**
```bash
helm dependency update
helm dependency list  # Verify dependencies are present
```

### Issue: ExternalSecret not syncing

**Solution:**
```bash
# Check ExternalSecret status
kubectl describe externalsecret azure-config -n monitoring

# Check ClusterSecretStore
kubectl get clustersecretstore azure-backend
kubectl describe clustersecretstore azure-backend

# Verify External Secrets Operator is running
kubectl get pods -n external-secrets-system
```

### Issue: Pods in CrashLoopBackOff

**Solution:**
```bash
# Check pod logs
kubectl logs -n monitoring <pod-name>

# Check if secrets exist
kubectl get secrets -n monitoring

# Verify ConfigMap
kubectl describe configmap resourcemanager-exporter-cm -n monitoring
```

### Issue: ServiceMonitor not creating metrics

**Solution:**
```bash
# Verify Prometheus Operator is installed
kubectl get servicemonitors -n monitoring

# Check if Prometheus can reach the service
kubectl get svc -n monitoring

# Verify network policies aren't blocking access
kubectl get networkpolicies -n monitoring
```

## Environment-Specific Deployments

### Development

```bash
helm install monitoring-stack . \
  -n monitoring-dev \
  --create-namespace \
  -f values.yaml \
  -f values-dev.yaml
```

### Staging

```bash
helm install monitoring-stack . \
  -n monitoring-staging \
  --create-namespace \
  -f values.yaml \
  -f values-staging.yaml
```

### Production

```bash
helm install monitoring-stack . \
  -n monitoring \
  --create-namespace \
  -f values.yaml \
  -f values-production.yaml \
  --atomic \
  --timeout 10m
```

## Monitoring the Deployment

### Watch pod status during deployment

```bash
kubectl get pods -n monitoring -w
```

### Follow logs

```bash
# KeyVault Exporter
kubectl logs -f -n monitoring -l app.kubernetes.io/name=azure-keyvault-exporter

# Resource Manager Exporter
kubectl logs -f -n monitoring -l app.kubernetes.io/name=azure-resourcemanager-exporter
```

### Check metrics are being collected

```bash
# List all services
kubectl get svc -n monitoring

# Port forward and check metrics
kubectl port-forward -n monitoring svc/azure-kv-exporter 8080:8080
curl http://localhost:8080/metrics | grep azure_keyvault
```

## Next Steps

1. Configure Prometheus to scrape the ServiceMonitors
2. Create Grafana dashboards for visualization
3. Set up alerting rules in Prometheus
4. Configure backup for critical configurations
5. Document custom values for disaster recovery
