ARG RUBY_VERSION=4.0.9
FROM ghcr.io/rails/devcontainer/images/ruby:${RUBY_VERSION}

LABEL org.opencontainers.image.source="https://github.com/fishpercolator/devcontainers"

# libvips for Active Storage variants, Chromium for Cuprite-driven system tests. Video and PDF
# previews (ffmpeg, poppler-utils) are left to the projects that need them, as they're most of the
# Rails activestorage feature's size.
USER root
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y libvips chromium && \
    rm -rf /var/lib/apt/lists/*

# The native installer rather than the npm package, so the image doesn't need Node.
#
# The directories are the targets of the shared volumes described in the README. Creating them here
# means a new volume takes their vscode ownership on first mount instead of being owned by root.
USER vscode
RUN curl -fsSL https://claude.ai/install.sh | bash && \
    mkdir -p /home/vscode/.bun/install/cache /home/vscode/.gems /home/vscode/.config/gh /home/vscode/.claude

# Gems go in a shared volume rather than Ruby's own gem directory, which holds Ruby's default gems.
# Bundler keeps each Ruby ABI in its own subdirectory, so one volume serves every Ruby version.
# Clean is off because a `bundle clean` in one project would remove gems other projects use.
ENV BINDING="0.0.0.0" \
    BUNDLE_PATH="/home/vscode/.gems" \
    BUNDLE_CLEAN="false" \
    CLAUDE_CONFIG_DIR="/home/vscode/.claude"
