# GitHub Actions Workflow: Build and Push Docker Image to Azure Container Registry then-deploy

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

#### Option B: Using Service Principal (Recommended for Production)

```bash
# Create service principal with push permissions
az ad sp create-for-rbac --name "github-actions-acr" \
  --role AcrPush \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.ContainerRegistry/registries/{registry-name}
```

Use the output `appId` as `ACR_USERNAME` and `password` as `ACR_PASSWORD`.

### 3. Dockerfile

Ensure you have a `Dockerfile` in the repository root. Example:

```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

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

## Deployment on Bare Metal Server

After the Docker image is successfully built and pushed to ACR, you can deploy it on a bare metal server or VM.

### Prerequisites for Bare Metal Deployment

- A Linux server (Ubuntu, RHEL, Fedora, or similar)
- Docker installed on the server
- Network access to Azure Container Registry
- Port 8080 available (or your application's required port)

### Step-by-Step Deployment Guide

#### 1. Install Azure CLI

The Azure CLI is required to authenticate with Azure Container Registry.

**For Debian/Ubuntu:**

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**For RHEL/Fedora/CentOS:**

```bash
# Import Microsoft repository key
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# Add Azure CLI repository
sudo tee /etc/yum.repos.d/azure-cli.repo <<EOF
[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

# Install Azure CLI
sudo dnf install -y azure-cli
```

#### 2. Login to Azure

Authenticate with your Azure account:

```bash
az login
```

This will open a browser window for authentication. Follow the prompts to sign in.

**For servers without a browser (headless):**

```bash
az login --use-device-code
```

Follow the instructions to authenticate using a device code on another machine.

#### 3. Login to Azure Container Registry

Authenticate Docker with your ACR:

```bash
# Login to ACR
az acr login --name petclinic1

# Verify ACR access
az acr list -o table
```

**Expected Output:**
```
NAME         RESOURCE GROUP    LOCATION    SKU    LOGIN SERVER
petclinic1   myResourceGroup   eastus      Basic  petclinic1-f0fcc4a0emg4bcay.azurecr.io
```

#### 4. Pull Docker Image from ACR

Pull the specific image version you want to deploy:

```bash
# Pull by build number (example: build #32)
docker pull petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32

# Or pull by commit SHA
docker pull petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:abc123def456

# Verify image is downloaded
docker images
```

**Expected Output:**
```
REPOSITORY                                          TAG       IMAGE ID       CREATED         SIZE
petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic   32        1a2b3c4d5e6f   2 hours ago     450MB
```

#### 5. Run Docker Container

Deploy the application by running the Docker container:

```bash
# Run container in detached mode
docker run -d \
  --name petclinic \
  -p 8080:8080 \
  --restart unless-stopped \
  petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32
```

**Options Explained:**
- `-d` - Run in detached mode (background)
- `--name petclinic` - Assign container name
- `-p 8080:8080` - Map host port 8080 to container port 8080
- `--restart unless-stopped` - Auto-restart container on failure or server reboot

#### 6. Verify Deployment

Check that the container is running:

```bash
# List running containers
docker container ls

# Or use docker ps
docker ps
```

**Expected Output:**
```
CONTAINER ID   IMAGE                                                    STATUS         PORTS                    NAMES
f2e7a1b3c4d5   petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32     Up 2 minutes   0.0.0.0:8080->8080/tcp   petclinic
```

#### 7. View Application Logs

Monitor the application logs to ensure it's running correctly:

```bash
# View logs (last 100 lines)
docker logs f2e7

# Follow logs in real-time (Ctrl+C to exit)
docker logs f2e7 -f

# Or use container name
docker logs petclinic -f
```

#### 8. Test Application

Access the application:

```bash
# Test from the server
curl http://localhost:8080

# Check application health endpoint (if available)
curl http://localhost:8080/actuator/health
```

From a browser, navigate to: `http://your-server-ip:8080`

### Complete Deployment Script

Create a deployment script for easy updates:

```bash
#!/bin/bash
# deploy-petclinic.sh

set -e  # Exit on error

# Configuration
ACR_NAME="petclinic1"
ACR_LOGIN_SERVER="petclinic1-f0fcc4a0emg4bcay.azurecr.io"
IMAGE_NAME="petclinic"
BUILD_NUMBER=$1
CONTAINER_NAME="petclinic"
HOST_PORT="8080"
CONTAINER_PORT="8080"

# Validate input
if [ -z "$BUILD_NUMBER" ]; then
    echo "Usage: ./deploy-petclinic.sh <build_number>"
    echo "Example: ./deploy-petclinic.sh 32"
    exit 1
fi

echo "=== Deploying PetClinic Build #$BUILD_NUMBER ==="

# Login to Azure and ACR
echo "Logging in to Azure..."
az acr login --name $ACR_NAME

# Pull the new image
echo "Pulling image: $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER"
docker pull $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER

# Stop and remove existing container (if exists)
echo "Stopping existing container..."
docker stop $CONTAINER_NAME 2>/dev/null || true
docker rm $CONTAINER_NAME 2>/dev/null || true

# Run new container
echo "Starting new container..."
docker run -d \
  --name $CONTAINER_NAME \
  -p $HOST_PORT:$CONTAINER_PORT \
  --restart unless-stopped \
  $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER

# Wait for container to start
sleep 5

# Verify deployment
echo "Verifying deployment..."
if docker ps | grep -q $CONTAINER_NAME; then
    echo "✓ Container is running"
    docker logs $CONTAINER_NAME --tail 20
    echo ""
    echo "=== Deployment Successful ==="
    echo "Access the application at: http://$(hostname -I | awk '{print $1}'):$HOST_PORT"
else
    echo "✗ Container failed to start"
    docker logs $CONTAINER_NAME
    exit 1
fi
```

**Make the script executable:**

```bash
chmod +x deploy-petclinic.sh
```

**Run the deployment:**

```bash
./deploy-petclinic.sh 32
```

### Managing the Deployed Application

#### Update to a New Version

```bash
# Stop current container
docker stop petclinic

# Remove old container
docker rm petclinic

# Pull new version
docker pull petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:33

# Run new version
docker run -d --name petclinic -p 8080:8080 --restart unless-stopped \
  petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:33
```

#### Rollback to Previous Version

```bash
# Stop current container
docker stop petclinic && docker rm petclinic

# Run previous version
docker run -d --name petclinic -p 8080:8080 --restart unless-stopped \
  petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32
```

#### View Container Resource Usage

```bash
# Real-time stats
docker stats petclinic

# One-time stats
docker stats --no-stream petclinic
```

#### Access Container Shell

```bash
# For debugging
docker exec -it petclinic /bin/bash

# Or sh if bash is not available
docker exec -it petclinic /bin/sh
```

#### Clean Up Old Images

```bash
# List all petclinic images
docker images | grep petclinic

# Remove specific old image
docker rmi petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:30

# Remove all unused images
docker image prune -a
```

### Firewall Configuration

If using a firewall, open the required port:

**For UFW (Ubuntu/Debian):**

```bash
sudo ufw allow 8080/tcp
sudo ufw reload
```

**For firewalld (RHEL/Fedora/CentOS):**

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Setting Up as a System Service (Optional)

For better control and automatic startup, create a systemd service:

```bash
sudo tee /etc/systemd/system/petclinic.service <<EOF
[Unit]
Description=PetClinic Docker Container
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/usr/bin/docker pull petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:latest
ExecStart=/usr/bin/docker run -d --name petclinic -p 8080:8080 --restart unless-stopped petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:latest
ExecStop=/usr/bin/docker stop petclinic
ExecStopPost=/usr/bin/docker rm petclinic

[Install]
WantedBy=multi-user.target
EOF
```

**Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable petclinic.service
sudo systemctl start petclinic.service
sudo systemctl status petclinic.service
```

### Troubleshooting Bare Metal Deployment

#### Cannot Pull Image - Authentication Failed

```bash
# Re-authenticate
az acr login --name petclinic1

# Or use docker login directly
docker login petclinic1-f0fcc4a0emg4bcay.azurecr.io
```

#### Port Already in Use

```bash
# Find process using port 8080
sudo lsof -i :8080

# Or
sudo netstat -tulpn | grep 8080

# Kill the process or use a different port
docker run -d --name petclinic -p 8081:8080 petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32
```

#### Container Exits Immediately

```bash
# Check container logs
docker logs petclinic

# Run interactively for debugging
docker run -it --rm petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32 /bin/bash
```

#### Out of Disk Space

```bash
# Check disk usage
df -h

# Clean up Docker resources
docker system prune -a --volumes
```

### Monitoring and Health Checks

Add health checks to your deployment:

```bash
docker run -d \
  --name petclinic \
  -p 8080:8080 \
  --restart unless-stopped \
  --health-cmd="curl -f http://localhost:8080/actuator/health || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  petclinic1-f0fcc4a0emg4bcay.azurecr.io/petclinic:32

# Check health status
docker inspect --format='{{.State.Health.Status}}' petclinic
```

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Azure Container Registry Documentation](https://docs.microsoft.com/azure/container-registry/)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
- [Azure CLI Documentation](https://docs.microsoft.com/cli/azure/)
- [Docker Run Reference](https://docs.docker.com/engine/reference/run/)

## License

Customize this section based on your project's license.

---

**Maintained by:** [Yakub Ilyas]  
**Last Updated:** January 2026
for project delivery contact : yakubiliyas12@gmail.com

---


