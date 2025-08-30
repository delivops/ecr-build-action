[![DelivOps banner](https://raw.githubusercontent.com/delivops/.github/main/images/banner.png?raw=true)](https://delivops.com)

# ECR Build and Push GitHub Action

![Docker](https://img.shields.io/badge/Docker-ECR-blue)

**Build and optionally push Docker images to AWS ECR, with support for multi-platform, caching, and tagging.**

This GitHub Action automates:
- Building Docker images using `docker/build-push-action`
- Tagging images with one or multiple tags
- Logging in to Amazon ECR
- Optionally pushing images to ECR

---

## How it works

This GitHub Action runs the following steps:

1. Optionally logs into AWS and Amazon ECR.
2. Ensures the target ECR repository exists.
3. Prepares tags and build arguments.
4. Sets up Docker Buildx (and QEMU if multi-arch).
5. Builds and optionally pushes the Docker image to ECR.

---

## Features

✔️ **Multi-tag and Multi-platform Docker Builds**  
Build images for `linux/amd64`, `linux/arm64`, and more, with multiple tags.

✔️ **Docker Layer Caching**  
Faster builds using GitHub Actions cache.

✔️ **Build Args, Target Stage & Dockerfile Path Support**  
Flexible customization for various Docker build needs.

✔️ **`pull` and `no_cache` Controls**  
- `pull`: always refresh base images to avoid stale builds  
- `no_cache`: force rebuild without cache when needed  

---

## Inputs

| Name                 | Description                                                  | Required | Default        |
|----------------------|--------------------------------------------------------------|----------|----------------|
| `image_name`         | Name of the Docker image to build and push                   | ✅       | —              |
| `tag`                | Comma-separated list of tags to apply                        | ✅       | —              |
| `path`               | Path to the Docker context                                   | ❌       | `.`            |
| `build_args`         | Comma-separated list of build arguments                      | ❌       | `""`           |
| `push`               | Whether to push the Docker image to ECR                      | ❌       | `true`         |
| `force_ecr_login`    | Force a new login to ECR                                     | ❌       | `false`        |
| `docker_layer_cache` | Use Docker layer caching                                     | ❌       | `true`         |
| `target`             | Docker build target stage                                    | ❌       | `""`           |
| `dockerfile_path`    | Path to the Dockerfile                                       | ❌       | `Dockerfile`   |
| `platforms`          | Target platforms for Docker build                            | ❌       | `linux/amd64`  |
| `aws_account_id`     | AWS account ID                                               | ✅       | —              |
| `aws_region`         | AWS region                                                   | ✅       | —              |
| `aws_role`           | IAM role to assume                                           | ❌       | `github_services` |
| `pull`               | Always attempt to pull referenced images                     | ❌       | `true`         |
| `no_cache`           | Do not use cache when building the image                     | ❌       | `false`        |

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
      - uses: delivops/ecr-build-push-action@v0.1.0
        with:
          image_name: "my-app"
          tag: "latest,sha-${{ github.sha }}"
          build_args: "ENV=production,VERSION=${{ github.sha }}"
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          aws_region: ${{ secrets.AWS_DEFAULT_REGION }}

## Notes

- This action uses `aws-actions/amazon-ecr-login` and `docker/build-push-action` internally.
- Make sure the IAM role (default: `github_services`) has permission to interact with ECR.
