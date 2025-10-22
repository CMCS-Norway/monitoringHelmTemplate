# Release Guide

This chart uses **automated versioning** - the version in `Chart.yaml` is automatically updated from git tags.

## 🚀 Creating a New Release

### Simple 3-Step Process

```bash
# 1. Make your changes and commit
git add .
git commit -m "feat: add new monitoring feature"
git push origin main

# 2. Create a version tag (the version you want)
git tag -a v1.0.1 -m "Release v1.0.1 - Bug fixes and improvements"

# 3. Push the tag (triggers automatic release)
git push origin v1.0.1
```

**That's it!** The workflow will:
- ✅ Extract version from tag (`v1.0.1` → `1.0.1`)
- ✅ Update `Chart.yaml` automatically
- ✅ Package chart as `monitoring-stack-1.0.1.tgz`
- ✅ Create GitHub Release
- ✅ Publish to Helm repository

## 📋 Version Numbering (Semantic Versioning)

Use semantic versioning for your tags:

- **Patch** (`v1.0.0` → `v1.0.1`): Bug fixes, minor updates
- **Minor** (`v1.0.0` → `v1.1.0`): New features, backward compatible
- **Major** (`v1.0.0` → `v2.0.0`): Breaking changes

## 🔍 What Happens During Release

```mermaid
graph LR
    A[Push tag v1.0.1] --> B[Workflow triggered]
    B --> C[Extract version: 1.0.1]
    C --> D[Update Chart.yaml]
    D --> E[Package chart]
    E --> F[Create GitHub Release]
    F --> G[Publish to Helm repo]
```

## ⚡ Quick Release Commands

### Patch Release (Bug Fixes)
```bash
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1
```

### Minor Release (New Features)
```bash
git tag -a v1.1.0 -m "Release v1.1.0 - New exporters added"
git push origin v1.1.0
```

### Major Release (Breaking Changes)
```bash
git tag -a v2.0.0 -m "Release v2.0.0 - Breaking: New configuration structure"
git push origin v2.0.0
```

## 🔄 Release Workflow

1. **Development**
   ```bash
   git checkout -b feature/new-exporter
   # Make changes
   git commit -m "feat: add postgres exporter"
   git push origin feature/new-exporter
   ```

2. **Merge to Main** (via PR)
   ```bash
   # PR is reviewed and merged
   ```

3. **Create Release**
   ```bash
   git checkout main
   git pull origin main
   git tag -a v1.2.0 -m "Release v1.2.0 - Add PostgreSQL exporter"
   git push origin v1.2.0
   ```

4. **Verify Release** (~2 minutes later)
   - Check: https://github.com/CMCS-Norway/monitoringHelmTemplate/releases
   - Test: `helm repo update && helm search repo cmcs-monitoring`

## 📝 Release Notes Template

```bash
git tag -a v1.2.0 -m "Release v1.2.0

New Features:
- Add PostgreSQL exporter support
- Enhanced monitoring for Azure SQL

Bug Fixes:
- Fix Fortigate scrape timeout
- Correct blackbox probe configuration

Dependencies:
- Update azure-keyvault-exporter to 1.0.13
"

git push origin v1.2.0
```

## ❌ Deleting/Fixing a Release

If you made a mistake:

```bash
# Delete tag locally and remotely
git tag -d v1.0.1
git push origin :refs/tags/v1.0.1

# Delete the GitHub Release manually (via web UI)
# Then create the correct tag
git tag -a v1.0.1 -m "Release v1.0.1 (corrected)"
git push origin v1.0.1
```

## 🎯 Pre-Release Checklist

Before creating a tag:

- [ ] All changes merged to `main`
- [ ] Tests passing (`helm lint`, `helm template`)
- [ ] Documentation updated (README, HELM_VALUES.md)
- [ ] CHANGELOG updated (if maintained)
- [ ] Dependencies up to date

## 🤖 Automated Processes

What's automated:
- ✅ Chart.yaml version update
- ✅ Chart packaging
- ✅ GitHub Release creation
- ✅ Helm repository publishing
- ✅ Dependency updates (via Renovate)

What requires manual action:
- ⚠️ Deciding version number
- ⚠️ Creating git tag
- ⚠️ Writing release notes

## 🔗 Related Documentation

- [Chart.yaml Specification](https://helm.sh/docs/topics/charts/#the-chartyaml-file)
- [Semantic Versioning](https://semver.org/)
- [GitHub Releases](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
