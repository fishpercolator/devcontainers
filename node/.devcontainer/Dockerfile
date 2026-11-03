FROM mcr.microsoft.com/devcontainers/base:trixie

LABEL org.opencontainers.image.source="https://github.com/fishpercolator/devcontainers"

# The native installer rather than the npm package, so Claude Code doesn't depend on the
# project's Node version.
#
# The directories are the targets of the shared volumes in devcontainer.json. Creating them here
# means a new volume takes their vscode ownership on first mount instead of being owned by root.
USER vscode
RUN curl -fsSL https://claude.ai/install.sh | bash && \
    mkdir -p /home/vscode/.bun/install/cache /home/vscode/.config/gh /home/vscode/.claude

ENV CLAUDE_CONFIG_DIR="/home/vscode/.claude"
