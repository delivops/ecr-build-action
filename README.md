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
2. Optionally logs into DockerHub to avoid pull rate limits on base images.
3. Ensures the target ECR repository exists.
4. Prepares tags and build arguments.
5. Sets up Docker Buildx (and QEMU if multi-arch).
6. Builds and optionally pushes the Docker image to ECR.

---

## Features

✔️ **Multi-tag and Multi-platform Docker Builds**  
Build images for `linux/amd64`, `linux/arm64`, and more, with multiple tags.

✔️ **Docker Layer Caching**
Faster builds using GitHub Actions cache.

✔️ **ECR Build Cache Support**
Push and reuse build layers stored alongside your image in Amazon ECR.

✔️ **Build Args, Target Stage & Dockerfile Path Support**  
Flexible customization for various Docker build needs.

✔️ **Skip Rebuilds of Existing Tags**
`skip_if_exists` reuses an image already in ECR instead of rebuilding and re-pushing it — required for immutable repositories.

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
| `skip_if_exists`     | Skip the build and push if the first tag already exists in ECR | ❌     | `false`        |
| `force_ecr_login`    | Force a new login to ECR                                     | ❌       | `false`        |
| `docker_layer_cache` | Use Docker layer caching (GitHub Actions cache)              | ❌       | `true`         |
| `enable_buildcache`  | Use an ECR registry cache for build layers                   | ❌       | `false`        |
| `buildcache_tag_name`| Tag used when storing build cache layers in ECR              | ❌       | `buildcache`   |
| `target`             | Docker build target stage                                    | ❌       | `""`           |
| `dockerfile_path`    | Path to the Dockerfile                                       | ❌       | `Dockerfile`   |
| `platforms`          | Target platforms for Docker build                            | ❌       | `linux/amd64`  |
| `aws_account_id`     | AWS account ID                                               | ✅       | —              |
| `aws_region`         | AWS region                                                   | ✅       | —              |
| `aws_role`           | IAM role to assume                                           | ❌       | `github_services` |
| `pull`               | Always attempt to pull referenced images                     | ❌       | `true`         |
| `no_cache`           | Do not use cache when building the image                     | ❌       | `false`        |
| `dockerhub_username` | DockerHub username for pulling base images                   | ❌       | `""`           |
| `dockerhub_access_token` | DockerHub access token for pulling base images           | ❌       | `""`           |
| `scan_on_push`       | Enable vulnerability scanning enforcement after image push   | ❌       | `false`        |
| `scan_severity_threshold` | Fail if vulnerabilities at or above this severity (CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL) | ❌ | `CRITICAL` |
| `scan_timeout`       | Maximum time in seconds to wait for scan results             | ❌       | `300`          |
| `scan_fail_on_timeout` | Whether to fail the action if scan times out               | ❌       | `true`         |

---

## Outputs

| Name      | Description                                                              |
|-----------|--------------------------------------------------------------------------|
| `skipped` | `"true"` when the tag already existed in ECR and nothing was built or pushed |

---

## 🚀 Usage

Pin an exact release. There is no floating `v0` tag — each release is tagged `v0.x.y` only, so
`@v0` does not resolve. The examples below use `v0.2.0`; check
[Releases](https://github.com/delivops/ecr-build-action/releases) for the latest.

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
      - uses: delivops/ecr-build-action@v0.2.0
        with:
          image_name: "my-app"
          tag: "latest,sha-${{ github.sha }}"
          build_args: "ENV=production,VERSION=${{ github.sha }}"
          enable_buildcache: "true"
          buildcache_tag_name: "buildcache"
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          aws_region: ${{ secrets.AWS_DEFAULT_REGION }}
          dockerhub_username: ${{ secrets.DOCKERHUB_USERNAME }}
          dockerhub_access_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

### With Vulnerability Scanning

```yaml
      - uses: delivops/ecr-build-action@v0.2.0
        with:
          image_name: "my-app"
          tag: "latest"
          scan_on_push: "true"
          scan_severity_threshold: "HIGH"
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          aws_region: ${{ secrets.AWS_DEFAULT_REGION }}
```

---

## Skipping Rebuilds & Immutable Repositories

ECR repositories configured for **tag immutability** reject a push to a tag that already
exists. That makes a re-run of a build workflow for an already-built commit fail, even though
the image it would produce is byte-identical to the one already there.

Set `skip_if_exists: "true"` and the action checks ECR before building. If the tag is already
present it skips both the build and the push, and reports `skipped: "true"`:

```yaml
      - id: build
        uses: delivops/ecr-build-action@v0.2.0
        with:
          image_name: "my-app"
          tag: "sha-${{ github.sha }}"
          skip_if_exists: "true"
          aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          aws_region: ${{ secrets.AWS_DEFAULT_REGION }}

      - if: steps.build.outputs.skipped == 'true'
        run: echo "Reused the image already in ECR"
```

### The first tag must be the unique one

The check reads the **first** tag in `tag`. It has to be one derived from the source —
`sha-${{ github.sha }}` — because "the tag exists" is taken to mean "the image built from this
exact source exists". With `tag: "latest,sha-${{ github.sha }}"` the check would look at
`latest`, find it always present, and never build anything again.

Write it the other way round:

```yaml
          tag: "sha-${{ github.sha }},latest"
```

Note that on a fully immutable repository a second tag like `latest` cannot be pushed twice
anyway — see the caveats below.

### Concurrent runs

Two runs building the same tag at the same time can both see "not in ECR yet" and both build.
The action handles this: when `skip_if_exists` is on, a push that loses the race is re-checked
against ECR, and if the tag is now present the step is treated as a success rather than a
failure. No configuration is needed for this.

If you would rather they never overlap at all, add a concurrency guard to the job that calls
this action:

```yaml
jobs:
  build:
    concurrency:
      group: build-my-app
      queue: max           # queue instead of replacing the pending run
    runs-on: ubuntu-latest
```

Put it on the job, not the workflow, so downstream deploy jobs are not serialized too. `queue:
max` matters: the default is `queue: single`, which **cancels** a pending run when a newer one
arrives — the second build would silently disappear rather than wait.

### Caveats on immutable repositories

- `enable_buildcache: "true"` rewrites the `buildcache_tag_name` tag on every build, so it
  cannot work against a fully `IMMUTABLE` repository. Either leave it off, or create the
  repository as `IMMUTABLE_WITH_EXCLUSION` with a `WILDCARD` exclusion filter covering that tag
  (and any other moving tag, such as `latest`). The default `docker_layer_cache` is unaffected
  — it uses the GitHub Actions cache, not ECR.
- The action creates missing repositories with ECR's default (mutable) settings. It never
  changes the mutability of an existing repository — manage that in Terraform.
- Skipping trusts the tag. The action cannot verify that the image already in ECR was built
  from your current source, which is exactly why the deciding tag must be content-derived.

### Additional IAM Permissions for Skipping

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:DescribeImages"
  ],
  "Resource": "*"
}
```

---

## 🔒 Vulnerability Scanning

When `scan_on_push` is enabled, the action will:
1. Wait for ECR to complete the vulnerability scan after pushing
2. Parse the scan findings by severity
3. Fail the workflow if vulnerabilities meet or exceed the threshold
4. Generate a summary table in the GitHub Actions UI

**Severity levels (from highest to lowest):** CRITICAL → HIGH → MEDIUM → LOW → INFORMATIONAL

Setting `scan_severity_threshold: "HIGH"` will fail if any CRITICAL or HIGH vulnerabilities are found.

### Additional IAM Permissions for Scanning

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:DescribeRepositories",
    "ecr:DescribeImageScanFindings",
    "ecr:StartImageScan"
  ],
  "Resource": "*"
}
```

---

## Notes

- This action uses `aws-actions/amazon-ecr-login` and `docker/build-push-action` internally.
- Make sure the IAM role (default: `github_services`) has permission to interact with ECR.
- When using the ECR build cache, either enable `push` or set `force_ecr_login: true` so the workflow can authenticate with ECR.
- **DockerHub login** is optional. When `dockerhub_username` and `dockerhub_access_token` are provided, the action authenticates with DockerHub before building. This avoids anonymous pull rate limits (100 pulls/6h unauthenticated vs 200 pulls/6h authenticated) and is recommended when your Dockerfile uses base images from DockerHub.
