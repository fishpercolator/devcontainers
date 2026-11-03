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
  If the volume does lose gems, `bin/setup` or `bundle install` in each project puts them back. Old
  versions therefore build up over time: see [Shared gems](#shared-gems) for clearing them out.
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
rebuilt image, the old one stays on disk until it's pruned (see below).

## Cleaning up disk space

Run these on the Docker host, or in a devcontainer with Docker access. Start by seeing where the
space has gone:

```sh
docker system df     # totals for images, containers, volumes and build cache
docker system df -v  # the same, item by item
```

### Routine tidy

These are safe to run at any time. They only remove things nothing is using, and everything they
remove can be rebuilt or downloaded again.

```sh
docker image prune    # images left untagged when a newer version was built or pulled
docker builder prune  # build cache not used by any image
```

### Deeper clean

When the routine tidy isn't enough. Each of these has a cost, given alongside it.

- **Images that no container uses:** `docker image prune -a`. This includes the base images of
  projects you haven't opened recently, and Selenium or Postgres images replaced by a newer tag,
  which are typically the biggest wins. The next rebuild of those projects downloads their images
  again.
- **Stopped containers:** `docker container prune`. A stopped devcontainer is usually just a
  project you don't have open, and its writable layer holds about 1 GB of caches and logs. With
  the shared volumes, nothing you need is lost: the project is rebuilt next time you open it.
  Shell history and anything else saved in the container itself outside the volumes is gone.
- **The VS Code server:** `docker volume rm vscode`, with every devcontainer stopped. The Dev
  Containers extension keeps every VS Code server version it has installed in this volume, and
  downloads the current one again next time it connects.
- **The bun cache:** `docker volume rm fishpercolator-bun-cache`, with every devcontainer stopped.
  bun downloads packages again on the next `bun install`.

### Shared gems

`fishpercolator-gems` keeps every gem version any project has installed, because `bundle clean`
can't be used on it. Expect it to grow by roughly 1–2 GB a year across the Rails projects.
When it's big, empty it, with the Rails devcontainers stopped:

```sh
docker volume rm fishpercolator-gems
```

Each project's next `bin/setup` (which runs when the devcontainer is rebuilt) or `bundle install`
installs just what its lockfile needs, at a cost of a few minutes per project. Gems are the only
thing in this volume, so nothing else is lost.

A Ruby minor-version upgrade (4.0 to 4.1, say) is a good time to do this anyway. The old version's
`ruby/<version>/` directory stops being used, unless another project is still on that version.
Patch upgrades (4.0.8 to 4.0.9) keep the same directory and reuse its gems.

### Volumes to leave alone

- **`fishpercolator-claude` and `fishpercolator-gh`** hold logins, and in Claude Code's case the
  memory and sessions for every project. Removing them can't be undone.
- **`<project>_postgres-data`** (and dump's `dump_valkey-data`) hold each project's development
  database. Removing one means running `bin/setup` (or `bin/rails db:setup`) in that project to
  recreate it, losing any data that isn't in its seeds.
- **`docker volume prune --all`** removes every volume no container is using, which includes all
  of the above for any project whose container has been removed. Use it only if you've checked
  `docker system df -v` first. Without `--all`, `docker volume prune` only removes anonymous
  volumes, which these setups don't create.

To stop every devcontainer, including the one you might be running these commands from:

```sh
docker stop $(docker ps -q --filter label=devcontainer.local_folder)
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
