[![DelivOps banner](https://raw.githubusercontent.com/delivops/.github/main/images/banner.png?raw=true)](https://delivops.com)

# ECR Build and Push GitHub Action

![Docker](https://img.shields.io/badge/Docker-ECR-blue)

**Build and optionally push Docker images to AWS ECR, with support for multi-platform, caching, and tagging.**

This GitHub Action automates:
- Building Docker images using `docker/build-push-action`
- Tagging images with one or multiple tags
- Logging in to Amazon ECR
- Optionally pushing images to ECR
- Skipping builds if the first tag already exists in the repository

---

## How it works

This GitHub Action runs the following steps:

1. Checks out the source code.
2. Optionally logs into AWS and Amazon ECR.
3. Skips the build if the first tag already exists in ECR.
4. Prepares tags and build arguments.
5. Sets up Docker Buildx for multi-platform builds.
6. Builds and optionally pushes the Docker image to ECR.

---

## Features

✔️ **Multi-tag and Multi-platform Docker Builds**  
Easily build images for `linux/amd64`, `linux/arm64`, and more, with multiple tags.

✔️ **Tag Existence Check**  
Skips unnecessary builds if the first tag already exists in the ECR repository.

✔️ **Docker Layer Caching**  
Faster builds using GitHub Actions cache.

✔️ **Build Args, Target Stage & Dockerfile Path Support**  
Flexible customization for various Docker build needs.

---

## Inputs

| Name                | Description                                                  | Required | Default           |
|---------------------|--------------------------------------------------------------|----------|-------------------|
| `image_name`        | Name of the Docker image to build and push                   | ✅       | —                 |
| `tag`               | Comma-separated list of tags to apply                        | ✅       | —                 |
| `path`              | Path to the Docker context                                   | ❌       | `.`               |
| `build_args`        | Comma-separated list of build arguments                      | ❌       | `""`              |
| `push`              | Whether to push the Docker image to ECR                      | ❌       | `true`            |
| `force_ecr_login`   | Force a new login to ECR                                     | ❌       | `false`           |
| `docker_layer_cache`| Use Docker layer caching                                     | ❌       | `true`            |
| `target`            | Docker build target stage                                    | ❌       | `""`              |
| `dockerfile_path`   | Path to the Dockerfile                                       | ❌       | `Dockerfile`      |
| `platforms`         | Target platforms for Docker build                            | ❌       | `linux/amd64`     |
| `runs_on`           | Runner image                                                 | ❌       | `ubuntu-latest`   |



---

## 🚀 Usage

```yaml
name: Build and Push Image

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: delivops/ecs-deploy-action@0.0.2
        with:
          image_name: "my-app"
          tag: "latest,sha-${{ github.sha }}"
          path: "./"
          build_args: "ENV=production,VERSION=${{ github.sha }}"
          push: true
          docker_layer_cache: true
          dockerfile_path: "./Dockerfile"
          platforms: "linux/amd64"
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          aws_region: ${{ secrets.AWS_DEFAULT_REGION }}
```

---

## Notes

- This action uses the `aws-actions/amazon-ecr-login` and `docker/build-push-action` internally.
- Make sure the IAM role `github_services` has permission to interact with ECR.
