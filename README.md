# devcontainers

Shared base images for Fish Percolator's devcontainers. Each project builds on one of these
instead of installing the same features itself, so the heavy layers are downloaded and stored
once per machine rather than once per project.

| Image | Contents | Used by |
|---|---|---|
| `ghcr.io/fishpercolator/devcontainer-rails:<ruby version>` | Rails devcontainer Ruby image, postgres-client 18, bun, gh, libvips, Chromium, Claude Code | shufflehorse, wdil, name.pn, dump |
| `ghcr.io/fishpercolator/devcontainer-node:24` | Debian trixie, Node 24, bun, gh, Claude Code | fuckingtomato |

Ruby images are built for each version in the `rails` job's matrix in
[`.github/workflows/build.yml`](.github/workflows/build.yml). Projects only share layers when
they're on the same Ruby, so prefer moving a project to a listed version over adding a new one.

## Using an image

```dockerfile
# .devcontainer/Dockerfile
ARG RUBY_VERSION=4.0.9
FROM ghcr.io/fishpercolator/devcontainer-rails:$RUBY_VERSION
```

Add only what that project needs on top, as features in its `devcontainer.json` or packages in its
`Dockerfile`. Anything added there sits above the shared layers, so it doesn't stop them being
shared.

To share bun's download cache between projects, mount the same named volume in each project's
`compose.yaml`:

```yaml
services:
  rails-app:
    volumes:
      - bun-cache:/home/vscode/.bun/install/cache

volumes:
  bun-cache:
    name: fishpercolator-bun-cache
```

## Rebuilds

Images rebuild on every push to `main` and on the first of each month. After a project picks up a
rebuilt image, the old one stays on disk until it's pruned:

```sh
docker image prune
```

## Docker host settings

Docker's build cache grows without limit by default. On each host, cap it in
`/etc/docker/daemon.json` and restart Docker:

```json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "5GB"
    }
  }
}
```
