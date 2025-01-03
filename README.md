# github-action-docker-bake

An action to build and publish docker images using docker bake

## Usage

This action assumes a `docker-bake.hcl` file is present in the root of the repository (the path can be overridden with the `file` input).

By default, the 'ci' target is called.

### basic docker-bake.hcl example

```hcl
group "default" {
  targets = ["app"]
}

target "app" {
  dockerfile = "Dockerfile"
  tags = ["ghcr.io/acme/app"]
}

group "ci" {
  targets = ["app-ci"]
}

# populated by the action, defines labels and tags to targets which inherit from it
target "docker-metadata-action" {}

target "app-ci" {
  inherits = ["app", "docker-metadata-action"]
}
```

### multi image with secrets example

```hcl
group "default" {
  targets = ["frontend", "backend"]
}

target "_common" {
  secret = [
    "id=github_oauth_token,env=GITHUB_OAUTH_TOKEN",
  ]
}

target "frontend" {
  inherits = ["_common"]
  dockerfile = "Dockerfile"
  target = "frontend"
  tags = ["acme/frontend"]
}

target "backend" {
  inherits = ["_common"]
  dockerfile = "Dockerfile"
  target = "backend"
  tags = ["acme/backend"]
}

#
# CI targets
#

group "ci" {
  targets = ["frontend-ci", "backend-ci"]
}

variable "PUBLISH_REPO" {
  default = "ghcr.io/acme/foo"
}

# populated by the action, defines labels and tags to targets which inherit from it
target "docker-metadata-action" {}

target "_common-ci" {
  inherits = ["docker-metadata-action"]
  platforms = ["linux/amd64", "linux/arm64"]
}

target "frontend-ci" {
  inherits = ["frontend", "_common-ci"]
  tags = [for tag in target.docker-metadata-action.tags : "${PUBLISH_REPO}/frontend:${tag}"]
  cache-from = ["type=gha,scope=frontend"]
  cache-to = ["type=gha,mode=max,scope=frontend"]
}

target "backend-ci" {
  inherits = ["backend", "_common-ci"]
  tags = [for tag in target.docker-metadata-action.tags : "${PUBLISH_REPO}/backend:${tag}"]
  cache-from = ["type=gha,scope=backend"]
  cache-to = ["type=gha,mode=max,scope=backend"]
}
```

and the secrets referencing environment variables can be provided when calling the action, e.g.

```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.pull_request.head.sha || '' }}

- name: Log in to the Container registry
  uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and Publish
  uses: briceburg/github-action-docker-bake@v1
  env:
    GITHUB_OAUTH_TOKEN: ${{ secrets.GITHUB_OAUTH_TOKEN }}
  with:
    force-push: ${{ inputs.force-push && 'true' || 'false' }}
```

### Inputs

| name | description | required | default |
| --- | --- | --- | --- |
| `cache-branches` | <p>List of branches that trigger publishing to cache. Defaults to the repository default branch.</p> | `false` | `${{ github.event.repository.default_branch }}` |
| `file` | <p>Path to Docker Bake file(s)</p> | `false` | `./docker-bake.hcl` |
| `force-push` | <p>When 'true', force push to registry even if current branch is not in push-branches</p> | `false` | `false` |
| `push-branches` | <p>List of branches that trigger publishing of images to registry. Defaults to the repository default branch.</p> | `false` | `${{ github.event.repository.default_branch }}` |
| `targets` | <p>List of bake targets</p> | `false` | `ci` |


### Outputs

| name | description |
| --- | --- |
| `metadata` | <p>Build result metadata</p> |
