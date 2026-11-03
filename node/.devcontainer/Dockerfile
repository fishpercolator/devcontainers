FROM mcr.microsoft.com/devcontainers/base:trixie

LABEL org.opencontainers.image.source="https://github.com/fishpercolator/devcontainers"

# The native installer rather than the npm package, so Claude Code doesn't depend on the
# project's Node version.
USER vscode
RUN curl -fsSL https://claude.ai/install.sh | bash && \
    mkdir -p /home/vscode/.bun/install/cache
