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
USER vscode
RUN curl -fsSL https://claude.ai/install.sh | bash && \
    mkdir -p /home/vscode/.bun/install/cache

ENV BINDING="0.0.0.0"
