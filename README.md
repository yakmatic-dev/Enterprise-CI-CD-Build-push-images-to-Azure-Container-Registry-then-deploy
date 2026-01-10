# Enterprise-CI-CD-Build-push-images-to-Azure-Container-Registry-then-deploy
# GitHub Actions Workflow: Build and Push Docker Image to Azure Container Registry

## Overview

This GitHub Actions workflow automates the process of building a Docker image and pushing it to Azure Container Registry (ACR) whenever code is pushed to the `main` branch. It includes optimized build caching using GitHub Actions cache to speed up subsequent builds.

## Workflow Trigger

```yaml
on:
  push:
    branches:
      - main
```

**Trigger Condition:** The workflow runs automatically on every push to the `main` branch.

## Workflow Jobs

### Job: `build-and-push`

**Purpose:** Build a Docker image from the repository and push it to Azure Container Registry with multiple tags.

**Runner:** `ubuntu-latest` - Uses the latest Ubuntu environment provided by GitHub.

## Workflow Steps

### 1. Checkout Repository

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

**Purpose:** Checks out the repository code so the workflow can access the Dockerfile and application source code.

**Action Used:** `actions/checkout@v4` - The latest stable version for checking out code.

### 2. Set Up Docker Buildx

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
  with:
    driver: docker-container
```

**Purpose:** Configures Docker Buildx, an extended build engine that enables advanced features like multi-platform builds and efficient caching.

**Why Required:** The GitHub Actions cache backend (`cache=gha`) requires Buildx to function properly.

**Driver:** Uses `docker-container` driver for isolated build environments.

### 3. Inspect Buildx (Optional Debug Step)

```yaml
- name: Inspect Buildx
  run: docker buildx inspect
```

**Purpose:** Displays information about the Buildx builder instance for debugging and verification purposes.

**Use Case:** Helpful for troubleshooting build issues or verifying the builder configuration.

### 4. Login to Azure Container Registry

```yaml
- name: Log in to Azure Container Registry
  uses: docker/login-action@v3
  with:
    registry: ${{ secrets.ACR_LOGIN_SERVER }}
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}
```

**Purpose:** Authenticates with Azure Container Registry to allow pushing Docker images.

**Required Secrets:**
- `ACR_LOGIN_SERVER` - Your ACR server URL (e.g., `myregistry.azurecr.io`)
- `ACR_USERNAME` - ACR username or service principal ID
- `ACR_PASSWORD` - ACR password or service principal password/token

### 5. Build and Push Docker Image

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: |
      ${{ secrets.ACR_LOGIN_SERVER }}/petclinic:${{ github.run_number }}
      ${{ secrets.ACR_LOGIN_SERVER }}/petclinic:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Purpose:** Builds the Docker image and pushes it to ACR with multiple tags while utilizing GitHub Actions cache for faster builds.

**Configuration:**
- **context:** `.` - Uses the repository root as the build context
- **file:** `./Dockerfile` - Specifies the Dockerfile location
- **push:** `true` - Automatically pushes the image to the registry after building
- **tags:** Creates two tags for each image:
  - `petclinic:RUN_NUMBER` - Sequential build number (e.g., `petclinic:42`)
  - `petclinic:COMMIT_SHA` - Full Git commit SHA (e.g., `petclinic:abc123def456...`)
- **cache-from:** `type=gha` - Pulls cache from GitHub Actions cache to speed up builds
- **cache-to:** `type=gha,mode=max` - Saves all build layers to GitHub Actions cache for maximum cache efficiency

## Prerequisites

### 1. Azure Container Registry Setup

You need an existing Azure Container Registry. If you don't have one:

```bash
# Create a resource group
az group create --name myResourceGroup --location eastus

# Create an ACR
az acr create --resource-group myResourceGroup \
  --name myregistry --sku Basic
```

### 2. GitHub Repository Secrets

Configure the following secrets in your GitHub repository:

**Navigate to:** Repository Settings → Secrets and variables → Actions → New repository secret

| Secret Name | Description | Example Value |
|-------------|-------------|---------------|
| `ACR_LOGIN_SERVER` | Your ACR login server URL | `myregistry.azurecr.io` |
| `ACR_USERNAME` | ACR username or service principal | `myregistry` or SP application ID |
| `ACR_PASSWORD` | ACR password or service principal password | Your ACR admin password or SP secret |

#### Option A: Using Admin Credentials

```bash
# Enable admin user on ACR
az acr update -n myregistry --admin-enabled true

# Get credentials
az acr credential show -n myregistry
```

#### Option B: Using Service Principal 

```bash
# Create service principal with push permissions
az ad sp create-for-rbac --name "github-actions-acr" \
  --role AcrPush \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.ContainerRegistry/registries/{registry-name}
```

Use the output `appId` as `ACR_USERNAME` and `password` as `ACR_PASSWORD`.

### 3. Dockerfile

Ensure you have a `Dockerfile` in the repository root.
## Image Tagging Strategy

The workflow creates two tags for each build:

1. **Run Number Tag:** `petclinic:42`
   - Incremental number for each workflow run
   - Useful for tracking builds sequentially
   - Example: `myregistry.azurecr.io/petclinic:42`

2. **Commit SHA Tag:** `petclinic:abc123def456...`
   - Full Git commit SHA
   - Allows tracing the exact code version
   - Example: `myregistry.azurecr.io/petclinic:abc123def456789`

### Accessing Images

After the workflow completes, pull images using:

```bash
# Pull by run number
docker pull myregistry.azurecr.io/petclinic:42

# Pull by commit SHA
docker pull myregistry.azurecr.io/petclinic:abc123def456789
```

## GitHub Actions Cache

The workflow uses GitHub Actions cache (`type=gha`) to significantly speed up builds:

- **First build:** Slower as it builds all layers and caches them
- **Subsequent builds:** Much faster by reusing cached layers that haven't changed
- **Cache mode:** `mode=max` - Caches all intermediate layers, not just the final image

**Cache Benefits:**
- Reduces build times by 50-90% for incremental changes
- Automatic cache invalidation when dependencies change
- Free cache storage (up to GitHub's limits)

## Customization Options

### Change Image Name

Replace `petclinic` with your application name:

```yaml
tags: |
  ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.run_number }}
  ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}
```

### Add Additional Tags

```yaml
tags: |
  ${{ secrets.ACR_LOGIN_SERVER }}/petclinic:${{ github.run_number }}
  ${{ secrets.ACR_LOGIN_SERVER }}/petclinic:${{ github.sha }}
  ${{ secrets.ACR_LOGIN_SERVER }}/petclinic:latest
```

### Trigger on Multiple Branches

```yaml
on:
  push:
    branches:
      - main
      - develop
      - release/*
```

### Different Dockerfile Location

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: ./app
    file: ./app/Dockerfile.prod
```

## Troubleshooting

### Authentication Failed

**Error:** `unauthorized: authentication required`

**Solution:** Verify your ACR credentials are correct in GitHub Secrets.

```bash
# Test ACR login locally
docker login myregistry.azurecr.io -u $ACR_USERNAME -p $ACR_PASSWORD
```

### Buildx Not Available

**Error:** `buildx is not available`

**Solution:** Ensure you're using a recent version of `docker/setup-buildx-action`.

### Cache Not Working

**Error:** Builds not using cache

**Solution:** 
- Verify Buildx is set up before the build step
- Check that `cache-from` and `cache-to` both use `type=gha`
- Ensure the repository has Actions cache enabled

### Image Not Found in ACR

**Error:** Cannot find pushed image

**Solution:**
- Check workflow logs for push confirmation
- Verify ACR_LOGIN_SERVER secret matches your registry
- List images in ACR:

```bash
az acr repository list --name myregistry --output table
az acr repository show-tags --name myregistry --repository petclinic
```

## Monitoring and Logs

### View Workflow Runs

Navigate to: Repository → Actions → Select workflow run

### Check ACR Repositories

```bash
# List all repositories in ACR
az acr repository list --name myregistry --output table

# Show tags for a specific repository
az acr repository show-tags --name myregistry --repository petclinic --output table

# Show image details
az acr repository show --name myregistry --repository petclinic:42
```

## Security Best Practices

1. **Use Service Principal:** Prefer service principals over admin credentials
2. **Least Privilege:** Grant only `AcrPush` role, not `Contributor` or `Owner`
3. **Rotate Credentials:** Regularly rotate ACR passwords and service principal secrets
4. **Enable Microsoft Defender:** Monitor ACR for vulnerabilities
5. **Use Azure RBAC:** Enable Azure RBAC for ACR instead of admin user in production

## Performance Metrics

Typical build times with caching:

- **First build (cold cache):** 5-10 minutes
- **Subsequent builds (warm cache, no changes):** 30-60 seconds
- **Incremental changes:** 1-3 minutes


## Project

for project delivery contact : yakubiliyas12@gmail.com

---

**Maintained by:** [Your Team/Organization]  
**Last Updated:** January 2026
