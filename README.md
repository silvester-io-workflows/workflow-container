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
    uses: silvester-io-workflows/workflow-container/.github/workflows/publish-image.yaml@v1
    with:
      runner: github # github (default), desktop, or homelab
      context: ./container
      dockerfile: ./container/containerfile
      platforms: linux/amd64
    secrets: inherit
```

### Runner selection and caching

`runner` accepts exactly three supported choices: `github`, `desktop`, or `homelab`. Omitting it preserves the original `ubuntu-latest` GitHub-hosted behavior. Runner selection is explicit, **not an automatic fallback**. The `desktop` runner needs to be online, and the `homelab` ARC scale set must have capacity.

| Choice | Target | Buildx driver / caching |
| --- | --- | --- |
| `github` (default) | `ubuntu-latest` | `docker/setup-buildx-action`, image-scoped GitHub Actions cache (`type=gha`, import and export) |
| `desktop` | `desktop` | Existing local Docker `default` builder; Docker image/cache layers persist on the host |
| `homelab` | `homelab` | Existing Docker `default` builder; image/cache layers persist on the shared Longhorn PVC across ephemeral ARC runner pods |

The `desktop` and `homelab` runners must provide a Docker daemon with Buildx enabled. The local default Docker driver is intended for the default `linux/amd64` image; multi-platform publishing may require a separately configured multi-platform-capable builder. These self-hosted modes deliberately do not upload Docker layers into GitHub's cache. Other workflow artifacts still use GitHub Actions `upload-artifact`/`download-artifact` regardless of runner type. Cache packages restored by `dotnet restore` run directly on the homelab runner are persisted separately at `/home/runner/.nuget/packages` using `NUGET_PACKAGES`; this does not automatically cache packages restored inside a Docker build. The shared runner PVC is disposable on cluster reprovisioning and must be used only by trusted jobs.

Example for the homelab ARC scale set:

```yaml
jobs:
  publish:
    uses: silvester-io-workflows/workflow-container/.github/workflows/publish-image.yaml@v1
    with:
      runner: homelab
      context: ./container
      dockerfile: ./container/containerfile
    secrets: inherit
```

### Inputs
- `runner` (optional): `github` (default), `desktop` or `homelab`.
- `registry` (optional): Registry hostname. Defaults to `ghcr.io`.
- `image_name` (optional): Image name to publish. Defaults to the caller repo (`owner/repo`).
- `context` (optional): Build context. Defaults to `./container`.
- `dockerfile` (optional): Containerfile/Dockerfile path. Defaults to `./container/containerfile`.
- `platforms` (optional): Target platforms. Defaults to `linux/amd64`.
- `artifact_name` (optional): download an artifact produced earlier in this workflow run before building (e.g. `cli`).
- `artifact_path` (optional): where to unpack it relative to the checkout (e.g. `.artifacts/cli/`); defaults to `.`.

A caller that compiles binaries separately can publish them using `actions/upload-artifact` in its validation job and provide the matching `artifact_name` and `artifact_path` to this reusable workflow. The publish job must depend on the validation job.


## Image tags

The calling repository chooses the triggers. The example above includes
`push.tags: ["v*"]`, so pushing a stable semantic-version tag such as `v1.2.3`
publishes all of these image tags for the same image:

- `:v1.2.3` — exact release version.
- `:v1.2` — rolling minor-version channel.
- `:v1` — rolling major-version channel.
- `:sha-<commit>` — commit-specific tag, also published on regular pushes.

Default-branch pushes continue to publish `:latest` and `:sha-<commit>`.
Releases **do not** update `:latest`. Non-semver Git tags retain their
previous exact-tag behavior. Pre-releases such as `v1.3.0-rc.1` publish only
their pre-release version and SHA tag, without moving stable version channels.

Major/minor tags deliberately move to the **most recently published**
release in that channel. Publish releases in version order: publishing an
older maintenance release later would otherwise move `:v1` backward.
Consumers requiring immutable references should pin an image digest.

The `@v1` in the `uses:` example selects this reusable workflow's branch;
it is separate from a container image tag such as `:v1`.
