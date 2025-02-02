_Note:_ Docker Scout is a new product and is free while in early access. Read more about [Docker Scout](https://www.docker.com/products/docker-scout?utm_source=hub&utm_content=scout-action-readme).

- [Docker Scout](#docker-scout)
- [Usage](#usage)
- [CLI Plugin Installation](#cli-plugin-installation)
- [Run as container](#run-as-container)
- [CI integration](#ci-integration)
- [License](#license)
 
# Docker Scout

[Docker Scout](https://www.docker.com/products/docker-scout/) is a collection of software supply chain features that appear throughout Docker user interfaces and the command line interface (CLI). These features provide detailed insights into the composition and security of container images.

This repository contains installable binaries of the `docker scout` CLI plugin.

## Usage

See the [reference documentation](https://docs.docker.com/scout) to learn about Docker Scout including Docker Desktop and Docker Hub integrations.

## CLI Plugin Installation

### Docker Desktop

`docker scout` CLI plugin is available by default on [Docker Desktop](https://docs.docker.com/desktop/) starting with version `4.17`.

### Script Installation

To install, run the following command in your terminal:

```shell
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
```

### Manual Installation

To install it manually:

- Download the `docker-scout` binary corresponding to your platform from the [latest](https://github.com/docker/scout-cli/releases/latest) or [other](https://github.com/docker/scout-cli/releases) releases.
- Uncompress it as
    - `docker-scout` on _Linux_ and _macOS_
    - `docker-scout.exe` on _Windows_
- Copy it in your local CLI plugin directory
    - `$HOME/.docker/cli-plugins` on _Linux_ and _macOS_
    - `%USERPROFILE%\.docker\cli-plugins` on _Windows_
- Make it executable on _Linux_ and _macOS_
    - `chmod +x $HOME/.docker/cli-plugins/docker-scout`
- Authorize the binary to be executable on _macOS_
    - `xattr -d com.apple.quarantine $HOME/.docker/cli-plugins/docker-scout`

## Run as container

A container image to run the Docker Scout CLI in containerized environments is available at [docker/scout-cli](https://hub.docker.com/r/docker/scout-cli). 

## CI Integration

Docker Scout CLI can be used in CI environments. See below for the various ways to integrate the CLI into your CI pipelines.

### GitHub Action

An early prototype of running the Docker Scout CLI as part of a GitHub Action workflow is available at [docker/scout-action](https://github.com/docker/scout-action).

The following GitHub Action workflow can be used as a template to integrate Docker Scout:

```yaml
name: Docker

on:
  push:
    tags: [ "*" ]
    branches:
      - 'main'
  pull_request:
    branches: [ "**" ]
    
env:
  # Use docker.io for Docker Hub if empty
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}
  SHA: ${{ github.event.pull_request.head.sha || github.event.after }}

jobs:
  build:

    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          ref: ${{ env.SHA }}
          
      - name: Setup Docker buildx
        uses: docker/setup-buildx-action@v2.5.0
        with:
          driver-opts: |
            image=moby/buildkit:v0.10.6

      # Login against a Docker registry except on PR
      # https://github.com/docker/login-action
      - name: Log into registry ${{ env.REGISTRY }}
        uses: docker/login-action@v2.1.0
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PAT }}

      # Extract metadata (tags, labels) for Docker
      # https://github.com/docker/metadata-action
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v4.4.0
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          labels: |
            org.opencontainers.image.revision=${{ env.SHA }}
          tags: |
            type=edge,branch=$repo.default_branch
            type=semver,pattern=v{{version}}
            type=sha,prefix=,suffix=,format=short
      
      # Build and push Docker image with Buildx (don't push on PR)
      # https://github.com/docker/build-push-action
      - name: Build and push Docker image
        id: build-and-push
        uses: docker/build-push-action@v4.0.0
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Docker Scount
        id: docker-scout
        if: ${{ github.event_name == 'pull_request' }}
        uses: docker/scout-action@dd36f5b0295baffa006aa6623371f226cc03e506
        with:
          command: cves
          image: ${{ steps.meta.outputs.tags }}
          only-severities: critical,high
          exit-code: true
```

### Gitlab

Use the following pipeline definition as a template to get Docker Scout integrated in Gitlab CI:

```yaml
docker-build:
  image: docker:latest
  stage: build
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    # Install curl and the Docker Scout CLI
    - |
      apk add --update curl
      curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- 
      apk del curl 
      rm -rf /var/cache/apk/* 
    # Login to Docker Hub required for Docker Scout CLI
    - docker login -u "$DOCKER_HUB_USER" -p "$DOCKER_HUB_PAT"
  script:
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    # Get a CVE report for the built image and fail the pipeline when critical or high CVEs are detected
    - docker scout cves "$CI_REGISTRY_IMAGE${tag}" --exit-code --only-severity critical,high
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  rules:
    - if: $CI_COMMIT_BRANCH
      exists:
        - Dockerfile
```

### Microsoft Azure DevOps Pipelines

Use the following pipeline definition as a template to get Docker Scout integrated in Azure DevOps Pipelines:

```yaml
trigger:
- main

resources:
- repo: self

variables:
  tag: '$(Build.BuildId)'
  image: 'vonwig/nodejs-service'

stages:
- stage: Build
  displayName: Build image
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: Docker@2
      displayName: Build an image
      inputs:
        command: build
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        repository: $(image)
        tags: |
          $(tag)
    - task: CmdLine@2
      displayName: Find CVEs on image
      inputs:
        script: |
          # Install the Docker Scout CLI
          curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
          # Login to Docker Hub required for Docker Scout CLI
          docker login -u $(DOCKER_HUB_USER) -p $(DOCKER_HUB_PAT)
          # Get a CVE report for the built image and fail the pipeline when critical or high CVEs are detected
          docker scout cves $(image):$(tag) --exit-code --only-severity critical,high
```          

## License

The Docker Scout CLI is licensed under the Terms and Conditions of the [Docker Subscription Service Agreement](https://www.docker.com/legal/docker-subscription-service-agreement/).
