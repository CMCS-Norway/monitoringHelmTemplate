# Helm Chart Verification Guide

## Pre-Deployment Validation

Run these checks before deploying to production.

### 1. Chart Structure Validation

```bash
# Verify chart structure
helm lint .

# Expected output: "1 chart(s) linted, 0 chart(s) failed"
# Note: Warning about missing dependencies is expected before running `helm dependency update`
```

### 2. Download Dependencies

```bash
# Download dependency charts
helm dependency update

# Verify dependencies are present
helm dependency list

# Expected output:
# NAME                              VERSION  REPOSITORY                              STATUS
# azure-keyvault-exporter          1.0.12   https://webdevops.github.io/helm-charts/ ok
# azure-resourcemanager-exporter   1.3.5    https://webdevops.github.io/helm-charts/ ok
```

### 3. Render Templates (Dry Run)

```bash
# Render all templates without deploying
helm template monitoring-stack . --debug > rendered.yaml

# Review the output
less rendered.yaml

# Check for common issues:
grep -i "error\|warning\|nil" rendered.yaml
```

### 4. Validate YAML Syntax

```bash
# Validate all rendered YAML is syntactically correct
helm template monitoring-stack . | kubectl apply --dry-run=client -f -

# Expected: No errors, all resources validated
```

### 5. Check Resource Counts

```bash
# Count resources that will be created
helm template monitoring-stack . | grep "^kind:" | sort | uniq -c

# Expected resources:
# - ConfigMap (1-2: resourcemanager-exporter-cm)
# - Deployment (2: kv-exporter, rm-exporter)
# - ExternalSecret (2-3: azure-config, azure-kv-config)
# - Service (2: kv-exporter, rm-exporter)
# - ServiceAccount (1-2: if enabled)
# - ServiceMonitor (1-2: if enabled)
```

### 6. Verify Values Structure

```bash
# Check values file is valid YAML
yamllint values.yaml

# Or use yq to validate
yq eval '.' values.yaml > /dev/null && echo "✓ Valid YAML"
```

### 7. Test Conditional Rendering

```bash
# Test with KeyVault Exporter disabled
helm template monitoring-stack . --set azureKeyVaultExporter.enabled=false | grep -c "azure-kv"
# Expected: 0 (no kv-exporter resources)

# Test with ResourceManager Exporter disabled
helm template monitoring-stack . --set azureResourceManagerExporter.enabled=false | grep -c "resourcemanager"
# Expected: 0 (no rm-exporter resources)
```

### 8. Verify Secrets References

```bash
# Check that all secret references are consistent
helm template monitoring-stack . | grep -A 5 "secretKeyRef" | grep "name:"

# Verify:
# - azure-config (global)
# - azure-kv-config (KeyVault exporter)
```

### 9. Check Label Consistency

```bash
# Extract all labels and verify consistency
helm template monitoring-stack . | grep -A 10 "labels:" | grep "customer:"

# All should show: customer: customer
```

### 10. Validate Namespaces

```bash
# Ensure all resources use the correct namespace
helm template monitoring-stack . | grep "namespace:" | sort | uniq

# All should be: monitoring (or your configured namespace)
```

## Post-Deployment Validation

After deploying with `helm install`, run these checks:

### 1. Helm Release Status

```bash
# Check release was installed successfully
helm status monitoring-stack -n monitoring

# View all releases in namespace
helm list -n monitoring
```

### 2. Pod Health

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Expected: All pods in "Running" state

# Check pod events for issues
kubectl get events -n monitoring --sort-by='.lastTimestamp'
```

### 3. ExternalSecrets Synchronization

```bash
# Check ExternalSecrets are synced
kubectl get externalsecrets -n monitoring

# Expected: All with "SecretSynced" status

# Detailed check
kubectl describe externalsecret azure-config -n monitoring
kubectl describe externalsecret azure-kv-config -n monitoring

# Verify secrets were created
kubectl get secrets -n monitoring
```

### 4. ConfigMap Validation

```bash
# Check ConfigMap exists
kubectl get configmap resourcemanager-exporter-cm -n monitoring

# Verify content
kubectl get configmap resourcemanager-exporter-cm -n monitoring -o yaml
```

### 5. Service Endpoints

```bash
# Check services have endpoints
kubectl get endpoints -n monitoring

# Each service should have at least one endpoint
```

### 6. ServiceMonitor Creation

```bash
# If Prometheus Operator is installed
kubectl get servicemonitors -n monitoring

# Expected: ServiceMonitors for enabled exporters
```

### 7. Test Metrics Endpoints

```bash
# Port-forward to KeyVault Exporter
kubectl port-forward -n monitoring svc/azure-kv-exporter 8080:8080 &

# Curl metrics endpoint
curl -s http://localhost:8080/metrics | head -20

# Expected: Prometheus-formatted metrics

# Kill port-forward
pkill -f "port-forward.*azure-kv-exporter"
```

### 8. Check Logs for Errors

```bash
# KeyVault Exporter logs
kubectl logs -n monitoring -l app.kubernetes.io/name=azure-keyvault-exporter --tail=50

# Resource Manager Exporter logs
kubectl logs -n monitoring -l app.kubernetes.io/name=azure-resourcemanager-exporter --tail=50

# Look for: Authentication success, no error messages
```

### 9. Verify Resource Limits

```bash
# Check resource requests/limits are applied
kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: requests={.resources.requests} limits={.resources.limits}{"\n"}{end}{"\n"}{end}'
```

### 10. Network Policy (if enabled)

```bash
# Check network policies
kubectl get networkpolicies -n monitoring

# Verify policies allow required traffic
kubectl describe networkpolicy -n monitoring
```

## Continuous Validation

### Automated Testing Script

```bash
#!/bin/bash
# save as: validate-deployment.sh

NAMESPACE="monitoring"
RELEASE="monitoring-stack"

echo "🔍 Validating Helm deployment..."

# Check Helm release
if helm status $RELEASE -n $NAMESPACE &>/dev/null; then
    echo "✓ Helm release found"
else
    echo "✗ Helm release not found"
    exit 1
fi

# Check pods
POD_COUNT=$(kubectl get pods -n $NAMESPACE --no-headers 2>/dev/null | wc -l)
RUNNING_PODS=$(kubectl get pods -n $NAMESPACE --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)

if [ "$POD_COUNT" -eq "$RUNNING_PODS" ] && [ "$POD_COUNT" -gt 0 ]; then
    echo "✓ All $RUNNING_PODS pods are running"
else
    echo "✗ Pod status: $RUNNING_PODS/$POD_COUNT running"
    kubectl get pods -n $NAMESPACE
    exit 1
fi

# Check ExternalSecrets
ES_COUNT=$(kubectl get externalsecrets -n $NAMESPACE --no-headers 2>/dev/null | wc -l)
if [ "$ES_COUNT" -gt 0 ]; then
    echo "✓ Found $ES_COUNT ExternalSecret(s)"
else
    echo "✗ No ExternalSecrets found"
    exit 1
fi

# Check Services
SVC_COUNT=$(kubectl get svc -n $NAMESPACE --no-headers 2>/dev/null | wc -l)
if [ "$SVC_COUNT" -gt 0 ]; then
    echo "✓ Found $SVC_COUNT service(s)"
else
    echo "✗ No services found"
    exit 1
fi

echo ""
echo "✅ All validation checks passed!"
```

Make executable and run:
```bash
chmod +x validate-deployment.sh
./validate-deployment.sh
```

## Troubleshooting Checklist

If validation fails, check these common issues:

- [ ] Dependencies downloaded? Run `helm dependency update`
- [ ] ClusterSecretStore exists? `kubectl get clustersecretstore azure-backend`
- [ ] External Secrets Operator running? `kubectl get pods -n external-secrets-system`
- [ ] Correct namespace specified? Check `-n` flag
- [ ] Values file syntax valid? Run `yamllint values.yaml`
- [ ] Azure credentials in secret backend? Verify with secret store admin
- [ ] Network policies blocking traffic? Check `kubectl get networkpolicies`
- [ ] Resource quotas exceeded? `kubectl describe resourcequota -n monitoring`

## Performance Validation

### Memory and CPU Usage

```bash
# Check resource usage after 5 minutes
kubectl top pods -n monitoring

# Compare against limits in values.yaml
kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.resources.limits}{"\n"}{end}{end}'
```

### Metrics Collection Rate

```bash
# Check scrape success rate in Prometheus
# Query: up{namespace="monitoring"}

# Verify metrics are increasing
# Port-forward and check metric timestamps
kubectl port-forward -n monitoring svc/azure-kv-exporter 8080:8080 &
curl -s http://localhost:8080/metrics | grep "^azure_keyvault_" | head -5
```

## Sign-Off Checklist

Before considering the deployment complete:

- [ ] All validation steps passed
- [ ] Pods are running and healthy
- [ ] ExternalSecrets are synced
- [ ] Metrics endpoints are accessible
- [ ] No error messages in logs
- [ ] ServiceMonitors created (if Prometheus Operator enabled)
- [ ] Resource usage within expected limits
- [ ] Documentation reviewed and accessible
- [ ] Rollback procedure tested in non-prod
- [ ] Monitoring alerts configured

## Emergency Rollback

If critical issues are found:

```bash
# Immediate rollback to previous version
helm rollback monitoring-stack -n monitoring

# Or uninstall completely
helm uninstall monitoring-stack -n monitoring
```

Always test rollback procedures in a non-production environment first!
