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

A project that needs nothing beyond what the image provides uses the image directly, with no
`Dockerfile`:

```yaml
# .devcontainer/compose.yaml
services:
  rails-app:
    # Keep the tag in step with .ruby-version.
    image: ghcr.io/fishpercolator/devcontainer-rails:4.0.9
```

A project that adds packages of its own builds on the image in a `Dockerfile`:

```dockerfile
# .devcontainer/Dockerfile
ARG RUBY_VERSION=4.0.9
FROM ghcr.io/fishpercolator/devcontainer-rails:$RUBY_VERSION

USER root
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y imagemagick && \
    rm -rf /var/lib/apt/lists/*
USER vscode
```

Devcontainer features go in the project's `devcontainer.json` either way. Anything a project adds
sits above the shared layers, so it doesn't stop them being shared.

A compose-based project can only build from a `Dockerfile` if it also has at least one devcontainer
feature. With no features, the devcontainer CLI builds the service with `docker compose build`, from
a Dockerfile it generates in `/tmp`, and compose's `buildx bake` refuses to read files outside the
build context:

```
additional privileges requested: pass "--allow=fs.read=/tmp/devcontainercli-.../Dockerfile-with-features" to grant requested privileges
```

With features, the CLI builds a different way, which doesn't hit this. So a project with
no features uses `image:`, and a project that needs its own packages but has no features installs
them through a feature instead of a `Dockerfile`, or asks for them to be added to the base image.

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
The shared volumes below are cleared from inside a devcontainer rather than removed. Docker won't
remove a volume that any container references, even a stopped one, so removing one would mean
deleting every devcontainer that mounts it first.

- **Old VS Code server versions:** the Dev Containers extension's `vscode` volume, mounted at
  `/vscode` in every devcontainer, gains a server version (about 640 MB) each time VS Code updates.
  If it has grown, close every devcontainer window but one, then from a terminal in that one:

  ```sh
  ls /vscode/vscode-server/bin/linux-x64/  # one directory per version
  ps -eo args | grep -oE '/vscode/vscode-server/bin/linux-x64/[0-9a-f]+' | sort -u  # the running one
  ```

  Delete every directory except the running one. Closing the other windows matters because a
  devcontainer still running an older version would lose its server. Its next connection would
  download it again.
- **The bun cache:** `bun pm cache prune` in any devcontainer removes packages older than 30 days,
  and `bun pm cache rm` empties it. bun downloads whatever it needs again on the next `bun install`.

### Shared gems

`fishpercolator-gems` keeps every gem version any project has installed, because `bundle clean`
can't be used on it. Expect it to grow by roughly 1–2 GB a year across the Rails projects.
When it's big, empty it from a terminal in any Rails devcontainer:

```sh
rm -rf ~/.gems/ruby
```

Then run `bundle install` in each Rails project you have open, which installs just what its lockfile
needs, at a cost of a few minutes per project. Until then, anything running in those projects (a
`bin/dev` server, a test run) fails to find its gems. Projects you don't have open reinstall theirs
when `bin/setup` runs at their next rebuild, or with `bundle install` when you open them. Gems are
the only thing in this volume, so nothing else is lost.

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
