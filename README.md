Reusable workflow for publishing container images to GHCR (or another registry).

## Usage

Add a workflow in the consuming repository that triggers on the events you want and calls this reusable workflow:

```yaml
name: Publish <image> Image to GHCR

on:
  push:
    branches: ["main"]
    tags: ["v*"]
  workflow_dispatch: {}

permissions:
  actions: read
  contents: read
  packages: write

jobs:
  publish:
    uses: silvester-io-homelab/pipeline-container/.github/workflows/publish-image.yaml@main
    with:
      context: ./container
      dockerfile: ./container/containerfile
      platforms: linux/amd64
    secrets: inherit
```

### Inputs
- `registry` (optional): Registry hostname. Defaults to `ghcr.io`.
- `image_name` (optional): Image name to publish. Defaults to the caller repo (`owner/repo`).
- `context` (optional): Build context. Defaults to `./container`.
- `dockerfile` (optional): Containerfile/Dockerfile path. Defaults to `./container/containerfile`.
- `platforms` (optional): Target platforms. Defaults to `linux/amd64`.
