# Helm Values Documentation

This document describes all configurable values for the monitoring-stack Helm chart.

## Table of Contents
- [Global Configuration](#global-configuration)
- [Azure KeyVault Exporter](#azure-keyvault-exporter)
- [Azure Resource Manager Exporter](#azure-resource-manager-exporter)

---

## Global Configuration

### `global.namespace`
**Type:** `string`  
**Default:** `monitoring`  
**Description:** Namespace where all resources will be deployed.

### `global.customer`
**Type:** `string`  
**Default:** `customer`  
**Description:** Customer identifier used in labels (short form).

### `global.customerLabel`
**Type:** `string`  
**Default:** `Customer Name`  
**Description:** Full customer name used in labels.

### `global.platform`
**Type:** `string`  
**Default:** `cloud`  
**Description:** Platform identifier for labeling.

### `global.provider`
**Type:** `string`  
**Default:** `azure`  
**Description:** Cloud provider identifier.

### `global.azureSecrets.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Enable creation of global Azure ExternalSecret.

### `global.azureSecrets.externalSecret.name`
**Type:** `string`  
**Default:** `azure-config`  
**Description:** Name of the global Azure ExternalSecret.

### `global.azureSecrets.externalSecret.refreshInterval`
**Type:** `string`  
**Default:** `12h`  
**Description:** How often to refresh secrets from the external secret store.

### `global.azureSecrets.externalSecret.secretStoreRef`
**Type:** `object`  
**Description:** Reference to the ClusterSecretStore.
```yaml
secretStoreRef:
  kind: ClusterSecretStore
  name: azure-backend
```

### `global.azureSecrets.externalSecret.remoteKeys`
**Type:** `object`  
**Description:** Mapping of secret keys to remote key names in the secret store.
```yaml
remoteKeys:
  clientId: azure-client-id-customer
  clientSecret: azure-client-secret-customer
  tenantId: azure-tenant-id-customer
  logAnalyticsWorkspace: azure-la-id-customer
  subscriptionId: azure-resourcegraph-subs-id-customer
```

---

## Azure KeyVault Exporter

### `azureKeyVaultExporter.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Enable/disable Azure KeyVault Exporter deployment.

### `azureKeyVaultExporter.fullnameOverride`
**Type:** `string`  
**Default:** `azure-kv-exporter`  
**Description:** Override the full name of the deployment.

### `azureKeyVaultExporter.replicas`
**Type:** `int`  
**Default:** `1`  
**Description:** Number of replicas for the deployment.

### `azureKeyVaultExporter.env`
**Type:** `object`  
**Description:** Environment variables passed to the exporter.
```yaml
env:
  AZURE_ENVIRONMENT: "AZUREPUBLICCLOUD"
  AZURE_SUBSCRIPTION_ID: "subscription-id-1 subscription-id-2"
  KEYVAULT_FILTER: "where tags['monitoring'] == 'true'"
```

### `azureKeyVaultExporter.extraEnv`
**Type:** `array`  
**Description:** Additional environment variables from secrets.
```yaml
extraEnv:
  - name: AZURE_TENANT_ID
    valueFrom:
      secretKeyRef:
        name: azure-kv-config
        key: AZURE_TENANT_ID
```

### `azureKeyVaultExporter.resources`
**Type:** `object`  
**Description:** CPU and memory resource requests/limits.
```yaml
resources:
  limits:
    cpu: 500m
    memory: 200Mi
  requests:
    cpu: 100m
    memory: 200Mi
```

### `azureKeyVaultExporter.prometheus.monitor.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Enable ServiceMonitor creation for Prometheus Operator.

### `azureKeyVaultExporter.prometheus.monitor.relabelings`
**Type:** `array`  
**Description:** Prometheus relabeling configuration for adding custom labels.

### `azureKeyVaultExporter.additionalResources.externalSecret.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Create ExternalSecret for KeyVault exporter credentials.

### `azureKeyVaultExporter.additionalResources.scrapeConfig.enabled`
**Type:** `bool`  
**Default:** `false`  
**Description:** Create ScrapeConfig for Prometheus Agent mode.

---

## Azure Resource Manager Exporter

### `azureResourceManagerExporter.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Enable/disable Azure Resource Manager Exporter deployment.

### `azureResourceManagerExporter.fullnameOverride`
**Type:** `string`  
**Default:** `azure-resourcemanager-exporter`  
**Description:** Override the full name of the deployment.

### `azureResourceManagerExporter.image.tag`
**Type:** `string`  
**Default:** `25.5.1`  
**Description:** Docker image tag for the exporter.

### `azureResourceManagerExporter.env`
**Type:** `object`  
**Description:** Environment variables for the exporter.
```yaml
env:
  AZURE_ENVIRONMENT: AZUREPUBLICCLOUD
  CONFIG: "config/config.yml"
```

### `azureResourceManagerExporter.existingConfigMap`
**Type:** `string`  
**Default:** `resourcemanager-exporter-cm`  
**Description:** Name of the ConfigMap containing exporter configuration.

### `azureResourceManagerExporter.resources`
**Type:** `object`  
**Description:** CPU and memory resource requests/limits.
```yaml
resources:
  requests:
    memory: 128Mi
    cpu: 100m
  limits:
    memory: 256Mi
    cpu: 200m
```

### `azureResourceManagerExporter.additionalResources.configMap.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Create ConfigMap with exporter configuration.

### `azureResourceManagerExporter.additionalResources.configMap.data.collectors`
**Type:** `object`  
**Description:** Configuration for various Azure collectors.
```yaml
collectors:
  general:
    scrapeTime: 24h
  costs:
    scrapeTime: 24h
    queries:
      - name: by_subscriptions
        timeFrames: [MonthToDate]
        valueField: PreTaxCost
        subscriptions: []
  graph:
    scrapeTime: 24h
  defender:
    scrapeTime: 24h
  resourceHealth:
    scrapeTime: 24h
  reservation:
    scrapeTime: 24h
    granularity: monthly
    fromDays: 30
```

---

## Common Helm Chart Values

Both exporters support standard Helm chart configurations including:

- `nodeSelector` - Node selection constraints
- `affinity` - Pod affinity rules
- `tolerations` - Pod tolerations
- `securityContext` - Pod security context
- `containerSecurityContext` - Container security context
- `serviceAccount` - Service account configuration
- `service` - Service configuration
- `netpol` - Network policy configuration

---

## Blackbox Exporter

### `blackboxExporter.enabled`
**Type:** `bool`  
**Default:** `true`  
**Description:** Enable/disable Blackbox Exporter deployment.

### `blackboxExporter.config.modules`
**Type:** `object`  
**Description:** Blackbox exporter probe modules configuration.
```yaml
config:
  modules:
    http_2xx:
      timeout: 10s
      http:
        valid_status_codes: [200, 301, 302, 307, 401, 403]
```

### `blackboxExporter.resources`
**Type:** `object`  
**Description:** CPU and memory resource requests/limits.
```yaml
resources:
  requests:
    cpu: 5m
    memory: 30Mi
  limits:
    cpu: 10m
    memory: 50Mi
```

### `blackboxExporter.service.port`
**Type:** `int`  
**Default:** `9115`  
**Description:** Service port for the Blackbox exporter.

---

## Additional Resources

Refer to the official Helm chart documentation for complete details on these values:
- [azure-keyvault-exporter](https://github.com/webdevops/helm-charts/tree/main/charts/azure-keyvault-exporter)
- [azure-resourcemanager-exporter](https://github.com/webdevops/helm-charts/tree/main/charts/azure-resourcemanager-exporter)
- [prometheus-blackbox-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-blackbox-exporter)
