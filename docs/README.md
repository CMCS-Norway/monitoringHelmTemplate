# Documentation Index

This directory contains comprehensive documentation for the monitoring-stack Helm chart.

## 📖 Available Guides

### [HELM_VALUES.md](./HELM_VALUES.md)
Complete reference of all Helm values with descriptions, types, and examples.
- Global configuration
- Azure KeyVault Exporter values
- Azure Resource Manager Exporter values
- Fortigate Exporter values
- Blackbox Exporter values

### [DEPLOYMENT.md](./DEPLOYMENT.md)
Step-by-step deployment procedures and checklists.
- Prerequisites validation
- Environment-specific deployments
- Common deployment scenarios
- Post-deployment verification

### [VERIFICATION.md](./VERIFICATION.md)
Testing and validation procedures.
- Pre-deployment validation
- Post-deployment checks
- Automated testing scripts
- Troubleshooting checklist

### [RELEASE_GUIDE.md](./RELEASE_GUIDE.md)
How to create new releases of this chart.
- Automated versioning from git tags
- Release workflow
- Semantic versioning guidelines
- Release checklist

## 🔗 External Documentation

- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Azure KeyVault Exporter](https://github.com/webdevops/helm-charts/tree/main/charts/azure-keyvault-exporter)
- [Azure Resource Manager Exporter](https://github.com/webdevops/helm-charts/tree/main/charts/azure-resourcemanager-exporter)
- [Prometheus Blackbox Exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-blackbox-exporter)
- [External Secrets Operator](https://external-secrets.io/)

## 📝 Contributing to Documentation

When updating documentation:

1. **Keep it current** - Update docs when changing code
2. **Use examples** - Include working code examples
3. **Be clear** - Write for users who may not know the system
4. **Test commands** - Ensure all commands actually work
5. **Link references** - Link to related sections and external docs

## 🎯 Quick Navigation

- Back to [Main README](../README.md)
- View [Chart Structure](../README.md#️-repository-structure)
- See [Quick Start](../README.md#-quick-start)
