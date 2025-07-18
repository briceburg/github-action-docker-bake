# github-action-docker-bake

Build and Publish Images with Docker Bake.

## Usage

This GitHub Action builds and publishes Docker images using [Docker Bake](https://docs.docker.com/build/bake/).

### Example Workflow

* assumes `docker-bake.hcl` exists in the repo root.
* assumes image will be published to ghcr.io.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest # or use an arm runner
    steps:
      - uses: actions/checkout@v4
      - name: Log in to the Container registry [for publishing images to ghcr.io]
        uses: docker/login-action@74a5d142397b4f367a81961eba4e8cd7edddf772 # v3.4.0
        with:
          registry: ghcr.io # change this if publishing to a different registry.
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: briceburg/github-action-docker-bake@v2
        with:
          ...
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
