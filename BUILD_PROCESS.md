# Building and Testing GitHub Branches with CI/CD

## Overview
This process allows you to build and test branches from external repositories (like customer hotfixes or PRs) without local build environment complications. Particularly useful when:
- Local builds fail due to platform differences (ARM64 Mac vs Linux AMD64)
- Projects have strict Linux/environment requirements
- You need to provide custom builds to clients quickly

## Architecture
```
GitHub Actions (Linux) → Builds branch → Pushes to GHCR → Client pulls image
```

## Prerequisites
- GitHub account
- Docker installed locally
- Access to GitHub Container Registry (ghcr.io)

---

## Step-by-Step Process

### 1. Create Build Repository

Create a dedicated repo for the build workflow:
```bash
# On GitHub: Create new repository
# Name: [project]-[feature]-build
# Visibility: Private (or public)
# Initialize with README

# Clone locally
git clone https://github.com/YOUR_USERNAME/[project]-build.git
cd [project]-build
```

### 2. Set Up Workflow Structure
```bash
# Create GitHub Actions directory
mkdir -p .github/workflows
```

### 3. Create Workflow File

Create `.github/workflows/build-[project].yml`:
```yaml
name: Build [Project] [Feature]

on:
  workflow_dispatch:  # Manual trigger
  push:
    branches: [ main ]

env:
  IMAGE_NAME: [project-name]
  SOURCE_REPO: [org/repo]
  SOURCE_BRANCH: [branch-name]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout source repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.SOURCE_REPO }}
          ref: ${{ env.SOURCE_BRANCH }}

      # Add language-specific setup (Go, Node, etc.)
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'  # Adjust version as needed

      # Build binary/artifacts
      - name: Build
        run: |
          # Project-specific build commands
          go build -o scripts/[binary-name] ./cmd/[entrypoint]

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        run: |
          IMAGE_TAG="ghcr.io/${{ github.repository_owner }}/${{ env.IMAGE_NAME }}:[tag]"
          docker buildx build \
            --platform linux/amd64 \
            --build-arg TARGETARCH=amd64 \
            -f [path/to/Dockerfile] \
            -t ${IMAGE_TAG} \
            --push \
            [build-context-path]
          
          echo "✅ Image: ${IMAGE_TAG}"
```

### 4. Commit and Push Workflow
```bash
git add .github/workflows/
git commit -m "Add build workflow for [feature]"
git push
```

### 5. Trigger Build

1. Go to GitHub repo → **Actions** tab
2. Select your workflow from the left sidebar
3. Click **"Run workflow"** button
4. Click green **"Run workflow"** button in dropdown
5. Wait for build to complete (green checkmark)

### 6. Make Package Public (if needed)

1. Go to: `https://github.com/YOUR_USERNAME?tab=packages`
2. Click on the package
3. Click **"Package settings"** (right sidebar)
4. Scroll to **"Danger Zone"** → **"Change visibility"** → **"Public"**

### 7. Share with Client/Team

Provide:
```
Image: ghcr.io/YOUR_USERNAME/[project]:[tag]
Platform: linux/amd64
Additional config: [any environment variables or flags needed]
```

---

## Common Adaptations

### For Different Languages

**Node.js:**
```yaml
- name: Set up Node
  uses: actions/setup-node@v4
  with:
    node-version: '20'

- name: Build
  run: |
    npm ci
    npm run build
```

**Python:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.11'

- name: Build
  run: |
    pip install -r requirements.txt
    # Build commands
```

### Multi-Architecture Builds
```yaml
- name: Build and push Docker image
  run: |
    docker buildx build \
      --platform linux/amd64,linux/arm64 \  # Multiple platforms
      -t ${IMAGE_TAG} \
      --push \
      .
```

### Multiple Tags
```yaml
- name: Build and push with multiple tags
  run: |
    BASE_TAG="ghcr.io/${{ github.repository_owner }}/${{ env.IMAGE_NAME }}"
    docker buildx build \
      --platform linux/amd64 \
      -t ${BASE_TAG}:latest \
      -t ${BASE_TAG}:${{ github.sha }} \
      -t ${BASE_TAG}:[feature-name] \
      --push \
      .
```

---

## Troubleshooting

### Build Fails - Dependency Issues

**Problem:** Missing dependencies or build tools  
**Solution:** Add installation steps in workflow:
```yaml
- name: Install dependencies
  run: |
    sudo apt-get update
    sudo apt-get install -y [package-name]
```

### Authentication Errors When Pulling

**Problem:** `unauthorized` or `denied` errors  
**Solution:** 
1. Create GitHub Personal Access Token:
   - Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Generate new token
   - Check: `read:packages`, `write:packages`
2. Login locally:
```bash
   echo "YOUR_TOKEN" | docker login ghcr.io -u YOUR_USERNAME --password-stdin
```

### Wrong Architecture

**Problem:** `no matching manifest for linux/arm64`  
**Solution:** Force platform when pulling:
```bash
docker pull --platform linux/amd64 ghcr.io/USERNAME/image:tag
```

### Can't Find Package

**Problem:** Package doesn't appear in GitHub  
**Solution:** 
- Check Actions → Workflow run → Look for errors
- Verify `packages: write` permission in workflow
- Check if push step actually executed

---

## Best Practices

1. **Use descriptive branch/tag names** - Makes it clear what's being tested
2. **Document required flags/config** - Include in README or workflow output
3. **Set expiration on test images** - Clean up after testing
4. **Use workflow_dispatch** - Manual triggers prevent accidental builds
5. **Add comments in workflow** - Future you will thank you
6. **Test locally first (if possible)** - Catch obvious issues before CI
7. **Version your workflows** - Keep a changelog of what changed

---

## Template Checklist

When setting up a new build workflow:

- [ ] Created dedicated build repository
- [ ] Set up `.github/workflows` directory
- [ ] Customized workflow for project's language/framework
- [ ] Set correct `SOURCE_REPO` and `SOURCE_BRANCH`
- [ ] Adjusted build commands for project structure
- [ ] Verified Dockerfile path is correct
- [ ] Tested workflow runs successfully
- [ ] Made package public (if needed)
- [ ] Documented image usage for client
- [ ] Shared image URL and required configuration

---

## Example: Coder Template Integration
```hcl
# In your Coder template

data "coder_parameter" "use_custom_envbuilder" {
  name        = "use_custom_envbuilder"
  description = "Use custom envbuilder image with submodule support"
  type        = "bool"
  default     = false
}

resource "coder_agent" "main" {
  env = {
    ENVBUILDER_GIT_URL = var.git_repo
    ENVBUILDER_GIT_CLONE_SUBMODULES = data.coder_parameter.use_custom_envbuilder.value ? "true" : "false"
  }
}

resource "docker_container" "workspace" {
  image = data.coder_parameter.use_custom_envbuilder.value ? 
    "ghcr.io/YOUR_USERNAME/envbuilder:submodule-fix" : 
    "ghcr.io/coder/envbuilder:latest"
}
```

---

## Clean Up

When testing is complete and feature is merged:

1. Archive or delete the build repository
2. Delete the package from GHCR (if no longer needed)
3. Update templates to use official images

---

## Future Improvements

- Add automated testing steps in workflow
- Set up notifications (Slack, email) on build completion
- Create reusable workflow templates
- Add caching for faster builds
- Implement semantic versioning for tags

---

## Related Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Buildx](https://docs.docker.com/build/buildx/)

---

**Last Updated:** [DATE]  
**Author:** Carolina Urrea  
**Use Case:** Building envbuilder branch 74-git-submodule-support