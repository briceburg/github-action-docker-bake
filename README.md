# github-action-docker-bake

Build and Publish Images with Docker Bake.

## Usage

This GitHub Action builds and publishes Docker images using [Docker Bake](https://docs.docker.com/build/bake/).

### Example Workflow

* assumes `docker-bake.hcl` exists in the repo root.
* assumes image will be published to ghcr.io.

```yaml
on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      publish:
        type: boolean
        default: true
        description: When true, publish image to GHCR.

concurrency:
  group: "docker-build-${{ github.head_ref || github.ref_name }}"
  cancel-in-progress: true

jobs:
  docker-bake:
    runs-on: ubuntu-24.04
    steps:
    - uses: actions/checkout@v4
    - name: Log in to the Container registry
      uses: docker/login-action@74a5d142397b4f367a81961eba4e8cd7edddf772 # v3.4.0
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - id: bake
      name: Bake Image (${{ inputs.push && 'push' || 'no-push' }})
      uses: briceburg/github-action-docker-bake@v2
      with:
        push: ${{ github.event_name == 'push' && 'true' || inputs.publish }}
        disable-cache: ${{ github.ref_name != github.event.repository.default_branch && 'true' || 'false' }}
```

### Example Bakefile

Include a `ci` target group that declares which ci-decorated images to build. Decorated images;

* inherit the docker-metadata-action target (which injects tags from [docker/metadata-action](https://github.com/docker/metadata-action))
* include cache configuration -- in this case using the [gha cache backend](https://docs.docker.com/build/cache/backends/gha/).

:zap: **tip:** leverage [bake variables](https://docs.docker.com/build/bake/variables/) and [matrix targets](https://docs.docker.com/build/bake/matrices/) to keep things DRY.

```hcl
# docker-bake.hcl

group "default" {
    targets = ["switchboard"]
}

group "ci" {
  targets = ["ci-switchboard"]
}

target "switchboard" {
    context    = "./switchboard/"
    tags       = ["ghcr.io/briceburg/radio-pad-switchboard:latest"]
}

target "ci-switchboard" {
  inherits   = ["switchboard", "docker-metadata-action"]
  cache-from = ["type=gha,scope=switchboard"]
  cache-to   = ["type=gha,scope=switchboard"]

  tags = [
    for tag in target.docker-metadata-action.tags :
    "ghcr.io/briceburg/radio-pad-switchboard:${tag}"
  ]
}

# augemented by docker/metadata-action to inject tags.
target "docker-metadata-action" {tags=[]}
```

## Inputs

Name | Description | Default
--- | --- | ---
`disable-cache` | When 'true', disable all caching. Typically 'true' when called from PRs/non-default branches to limit cache sprawl. | `'false'`
`files` | Newline-delimited docker bake file path(s). | `'./docker-bake.hcl'`
`load` | When 'true', built images are loaded into the docker daemon. | `'false'`
`push` | When 'true', built images are published to their registry. | `'true'`
`quiet` | When 'true', exclude build output from logs and only print errors. | `'false'`
`targets` | Space-delimited bake targets. | `'ci'`
`upload-build-record` | When 'true', upload the build record archive. | `'false'`

## Outputs

Name | Description
--- | ---
`metadata`| Build result metadata
`version` | Version (first 'fixed' [non-rolling] image tag)

## How it Works

* [docker/setup-qemu-action](https://github.com/docker/setup-qemu-action) enables [multi-platform](https://docs.docker.com/build/bake/reference/#targetplatforms) builds.
* [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) enables a pinned version of buildx for deterministic builds.
* [docker/metadata-action](https://github.com/docker/metadata-action) generates image tags and labels.
* [docker/bake-action](https://github.com/docker/bake-action) builds and publishes images. Inputs are passed through to this action.
