## CI/CD and Automated Releases

This repository can use GitHub Actions to automate versioning, Docker image publishing, and GitHub Releases whenever changes are pushed to the `main` branch.

The example workflow is named **Backend Release** and is intended for repositories that build and publish a Docker image from the repository root.

### Release workflow overview

The pipeline performs the following operations:

1. **Checkout:** Checks out the repository source code.
2. **Automatic versioning:** Determines the next version using the latest Git tag and the most recent commit message.
3. **Git tagging:** Creates and pushes the new version tag to the repository.
4. **Docker Buildx setup:** Prepares Docker Buildx for image builds.
5. **GHCR authentication:** Authenticates with GitHub Container Registry.
6. **Docker image publishing:** Builds the Docker image and pushes versioned and `latest` tags.
7. **GitHub Release:** Creates a GitHub Release and generates release notes.

### Workflow trigger

The workflow runs automatically when a commit is pushed to `main`.

```
on:
  push:
    branches:
      - main
```

The workflow declares the following repository permissions:

```
permissions:
  contents: write
  packages: write
```

- `contents: write` allows the workflow to push Git tags and create releases.
- `packages: write` allows it to publish container images to GHCR.

Grant only the permissions required by the workflow.

### Automatic semantic versioning

The workflow calculates a new version using the latest Git tag.

Versions follow the format:

```
vMAJOR.MINOR.PATCH
```

The default behavior increments the patch version. Commit messages can request a major or minor version increment.

| Commit message contains | Version change | Example |
| --- | --- | --- |
| `#major` | Increment major; reset minor and patch | `v1.4.2` → `v2.0.0` |
| `#minor` | Increment minor; reset patch | `v1.4.2` → `v1.5.0` |
| Neither marker | Increment patch | `v1.4.2` → `v1.4.3` |

For example:

```
git commit -m "Add breaking API changes #major"
git push origin main
```

Or:

```
git commit -m "Add a new infrastructure feature #minor"
git push origin main
```

Ordinary fixes increment the patch version:

```
git commit -m "Fix Nginx routing configuration"
git push origin main
```

If no previous version tag exists, the workflow starts from `v0.0.0`, making the first generated version `v0.0.1`.

The workflow also checks remote tags to avoid reusing an existing version. However, concurrent release runs can still race, so releases should be serialized or protected with a GitHub Actions concurrency group.

### Docker image publishing

The workflow uses Docker Buildx and the Docker Build Push Action to build and publish a container image.

The image is published to GitHub Container Registry:

```
ghcr.io/<owner>/<repository>:<version>
ghcr.io/<owner>/<repository>:latest
```

For example:

```
ghcr.io/example/impactive-backend:v1.2.3
ghcr.io/example/impactive-backend:latest
```

Replace the example owner and repository with the actual GitHub repository coordinates.

The workflow uses GitHub Actions caching to speed up subsequent builds:

```
cache-from: type=gha
cache-to: type=gha,mode=max
```

**Important:** The example builds using `context: .`, which assumes that a suitable Dockerfile exists at the repository root. If the repository contains only `compose.yaml` and infrastructure configuration, a Docker image cannot be built successfully this way without an appropriate Dockerfile and build context.

For this infrastructure repository, choose the image and build context deliberately. For example, the Nginx image may require a context of `./nginx`, depending on the contents of `nginx/Dockerfile`. Alternatively, publish application images from their respective application repositories.

### GitHub Container Registry authentication

The example authenticates to GHCR using the GitHub Actions actor and a repository secret:

```
- name: Login to GHCR
  uses: docker/login-action@v4
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GHCR_PAT }}
```

Configure the `GHCR_PAT` secret in the repository's GitHub Actions secrets.

The token must have the required package publishing permissions and be authorized to access the repository or organization. Use a fine-grained credential where supported, restrict its scope, and rotate it as appropriate.

Where possible, consider using the automatically provided `GITHUB_TOKEN` with `packages: write` instead of a long-lived personal access token. Verify package ownership and organization policies before changing the authentication method.

### GitHub Releases

After publishing the image, the workflow creates a GitHub Release associated with the generated version tag.

The release includes automatically generated release notes:

```
- name: Create GitHub Release
  uses: softprops/action-gh-release@v2
  with:
    tag_name: ${{ env.NEW_VERSION }}
    name: Release ${{ env.NEW_VERSION }}
    generate_release_notes: true
```

The workflow authenticates using `GITHUB_TOKEN`.

Each successful release should provide a version reference that can be used to identify the corresponding source revision and container image.

### Required GitHub configuration

Before enabling automated releases, verify the following:

- The workflow file is stored under `.github/workflows/` and has a `.yml` or `.yaml` extension.
- The workflow has permission to push tags and create releases.
- GHCR authentication is configured correctly.
- The repository contains a valid Dockerfile for the image being built.
- The image name and build context match the intended deployment artifact.
- The repository's branch protection and Actions policies permit the required operations.
- The workflow has been tested on a non-production branch or test repository.

### Troubleshooting

**Docker build fails:** Check that the Dockerfile exists at the configured build context and that all required files are available to the build.

**GHCR push is denied:** Check token permissions, package access, repository ownership, and the `packages: write` permission.

**Tag already exists:** Check existing tags and prevent overlapping release runs. Do not overwrite published release tags.

**Release creation fails:** Verify `contents: write`, the release token, and that the requested tag exists remotely.

**Unexpected version increment:** Inspect the latest tag and the exact commit message. The example examines the most recent commit message rather than all commits included in a push.