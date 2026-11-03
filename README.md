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

## Shared volumes

Anything kept in a container's home directory is lost when the container is rebuilt. These named
volumes keep it instead, and share it between every project on the machine:

| Volume | Mounted at | Holds |
|---|---|---|
| `fishpercolator-bun-cache` | `~/.bun/install/cache` | bun's package downloads |
| `fishpercolator-gems` | `~/.gems` (`BUNDLE_PATH`) | installed gems (Rails image only) |
| `fishpercolator-gh` | `~/.config/gh` | the gh login |
| `fishpercolator-claude` | `~/.claude` (`CLAUDE_CONFIG_DIR`) | the Claude Code login, settings, plugins, and each project's memory and sessions |

The images set the environment variables and create the mount points. Projects using the Node
image get the volumes from the image itself. Compose-based projects (all the Rails ones) must list
them in their own `compose.yaml`:

```yaml
services:
  rails-app:
    volumes:
      - bun-cache:/home/vscode/.bun/install/cache
      - gems:/home/vscode/.gems
      - gh:/home/vscode/.config/gh
      - claude:/home/vscode/.claude

volumes:
  bun-cache:
    name: fishpercolator-bun-cache
  gems:
    name: fishpercolator-gems
  gh:
    name: fishpercolator-gh
  claude:
    name: fishpercolator-claude
```

The Rails image can't supply these itself. The devcontainer CLI declares an image's volume mounts in
the compose file it generates without a `name:`, so compose prefixes each one with the project name
and no two projects share it. Declaring it `external` fails until the volume has been created by
hand.

### Caveats

- **Never run `bundle clean`.** Every project's gems are in the same directory, so cleaning one
  project's bundle deletes the gems every other project depends on. `BUNDLE_CLEAN=false` stops
  Bundler doing it automatically after `bundle install`, which its docs say becomes the default.
  If the volume does lose gems, `bin/setup` or `bundle install` in each project puts them back.
- **`gem list` doesn't show a project's gems**, because they're in `BUNDLE_PATH` rather than Ruby's
  own gem directory. Use `bundle list`, and run commands with `bundle exec` or the project's
  binstubs.
- **Ruby versions share the volume safely.** Bundler installs each Ruby ABI under its own
  `ruby/<version>/` subdirectory, so the 3.4 and 4.0 projects never use each other's gems. Native
  extensions are built against the base image's libraries, so after a major change to the base
  image, `bundle pristine` rebuilds them.
- **Every container sees the same credentials.** The gh token (with whatever scopes you granted it)
  and the Claude Code login are available in every devcontainer on the machine, including to code
  run by any project's gems or npm packages. Each container held credentials like these anyway, so
  the main change is that a compromised dependency in one project can now use a login you only
  made in another.
- **Shared files are shared, per-project files aren't.** Claude Code's login, settings and plugins
  are shared. Its memory and sessions are stored per project path (`projects/-workspaces-<name>/`),
  so each project keeps its own. Running Claude in several containers at once is the same as
  running several sessions on one machine.
- **Removing a volume logs every project out.** `docker volume rm fishpercolator-claude` (or a
  `docker volume prune --all` while no devcontainer is using it) loses the Claude login, memory and
  sessions for every project at once. Similarly, `fishpercolator-gh` holds the gh login, and
  `fishpercolator-gems` holds every installed gem.

### Moving existing containers onto the volumes

Before rebuilding a devcontainer that predates the volumes, copy its Claude Code and gh state into
them, or it's lost with the old container:

```sh
bin/migrate-to-shared-volumes                  # every devcontainer on this machine
bin/migrate-to-shared-volumes wdil-rails-app-1 # just these
```

Run it on the Docker host, or in a devcontainer with Docker access, while the old containers still
exist. Where two containers hold the same file, the first one copied wins. It never overwrites a
file already in a volume, so run it immediately before rebuilding: running it again later won't
pick up anything the old container changed since the first run. Gems aren't copied: the first
`bin/setup` reinstalls them into the volume.

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
