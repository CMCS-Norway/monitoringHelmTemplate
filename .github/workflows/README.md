# GitHub Workflows

This repository uses GitHub Actions for automated testing, releases, and dependency management.

## 📁 Workflows

### 1. `release.yml` - Automated Release Pipeline

**Trigger**: When a tag starting with `v` is pushed (e.g., `v1.0.1`)

**What it does**:

1. **Lint & Test Job**:

   - Checks out code
   - Sets up Helm
   - Updates chart dependencies
   - Lints the Helm chart
   - Templates the chart to validate it renders correctly
   - Validates YAML syntax with yamllint

2. **Release Job** (only runs if tests pass):
   - Extracts version from tag (e.g., `v1.0.1` → `1.0.1`)
   - Updates `Chart.yaml` with the version
   - Packages the Helm chart
   - Creates a GitHub Release with:
     - Release notes
     - Chart package (.tgz file)
   - Updates the `gh-pages` branch with new Helm repository index
   - Makes the chart available via Helm repository

**Usage**:

```bash
# Just push a tag!
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1
```

### 2. `prepare-release.yml` - Prepare Release (Manual)

**Trigger**: Manual workflow dispatch via GitHub Actions UI

**What it does**:

1. Validates the version format (must be SemVer: x.y.z)
2. Checks if the tag already exists
3. Updates `Chart.yaml` with the new version
4. Commits the change
5. Creates and pushes the tag
6. Triggers the release workflow automatically

**Usage**:

1. Go to **Actions** → **Prepare Release**
2. Click **Run workflow**
3. Enter version (e.g., `1.0.1`)
4. Click **Run workflow**

### 3. `renovate.yml` - Dependency Updates

**Trigger**:

- Scheduled: Every Monday at 8 AM UTC
- Manual: Via workflow dispatch

**What it does**:

- Runs Renovate bot to check for dependency updates
- Creates PRs for:
  - Helm chart dependencies (Azure exporters, Blackbox exporter)
  - GitHub Actions versions
  - Other dependencies defined in `renovate.json`

## 🔐 Required Permissions

The workflows require the following GitHub token permissions:

- `contents: write` - For creating releases and updating gh-pages
- `pages: write` - For updating GitHub Pages
- `id-token: write` - For OIDC authentication

These are automatically provided by `GITHUB_TOKEN`.

## 🎯 Typical Release Flow

### Simple Flow (Recommended)

```bash
# 1. Use GitHub Actions UI
Go to Actions → Prepare Release → Run workflow → Enter version

# That's it! Everything else is automated.
```

### Manual Flow

```bash
# 1. Make changes
git add .
git commit -m "feat: add new feature"
git push

# 2. Update version and create tag
sed -i 's/^version:.*/version: 1.0.1/' Chart.yaml
git add Chart.yaml
git commit -m "chore(release): bump version to 1.0.1"
git push

# 3. Create and push tag
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1

# 4. Watch the magic happen in Actions tab! ✨
```

## 🐛 Troubleshooting

### Release workflow fails

**Check**:

1. Is the tag format correct? (must start with `v`, e.g., `v1.0.1`)
2. Do you have GitHub Pages enabled in repository settings?
3. Check workflow logs in Actions tab

### Chart not appearing in Helm repository

**Solutions**:

1. Check if `gh-pages` branch exists and has `index.yaml`
2. Verify GitHub Pages is published from `gh-pages` branch
3. Wait a few minutes for GitHub Pages to deploy

### Can't push tags

**Check**:

- Do you have write permissions to the repository?
- Is branch protection blocking tag creation?

## 📚 More Information

- [Release Guide](../../docs/RELEASE_GUIDE.md) - Detailed release instructions
- [Deployment Guide](../../docs/DEPLOYMENT.md) - How to deploy the chart
- [Helm Chart Documentation](../../README.md) - Chart configuration and usage
