# Helm Repository Index

This is the Helm chart repository for monitoring-stack.

## Using this Helm Repository

Add this repository to your Helm installation:

```bash
helm repo add monitoring-stack https://cmcs-norway.github.io/monitoringHelmTemplate
helm repo update
```

## Available Charts

- **monitoring-stack**: A comprehensive monitoring solution with Azure exporters and Blackbox
  exporter

## Installation

```bash
# Install with default values
helm install my-monitoring monitoring-stack/monitoring-stack

# Install with custom values
helm install my-monitoring monitoring-stack/monitoring-stack -f custom-values.yaml

# Install specific version
helm install my-monitoring monitoring-stack/monitoring-stack --version 1.0.1
```

## Documentation

For detailed documentation, visit the
[main repository](https://github.com/CMCS-Norway/monitoringHelmTemplate).
