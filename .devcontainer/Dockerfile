# Changes from upstream Anthropic template (github.com/anthropics/claude-code/tree/main/.devcontainer):
#   - Base image changed from node:20 to ubuntu:24.04; Node.js 20 installed via NodeSource
#   - User changed from the image's default `node` to a created `vscode` user (UID/GID 1000)
#   - Added: bubblewrap, socat (apt packages — required for Claude Code sandbox mode)
#   - Added: @anthropic-ai/sandbox-runtime to npm global installs (seccomp filters for sandbox)
#   - Added: pre-configured ~/.claude/settings.json with enableWeakerNestedSandbox: true
#     (required for bubblewrap to work inside Docker; volume mount removed in devcontainer.json
#      so this file is not shadowed on first run)
#   - Added: UV installer + PATH for Python toolchain management (interpreter versions,
#     project dependencies, and virtual environments)
#   - Added: chezmoi installer (dotfiles manager, installed to ~/.local/bin)
#   - Note: user plugins (e.g. superpowers) are NOT baked in — they are installed via VS Code
#     dotfiles (chezmoi apply) which runs before the firewall (waitFor: postCreateCommand)

FROM ubuntu:24.04

ARG TZ
ENV TZ="$TZ"

ARG CLAUDE_CODE_VERSION=latest

# Non-root user setup
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=1000

# Install basic development tools and iptables/ipset
RUN apt-get update && apt-get install -y --no-install-recommends \
  bubblewrap \
  socat \
  less \
  git \
  procps \
  sudo \
  fzf \
  zsh \
  man-db \
  unzip \
  gnupg2 \
  gh \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  jq \
  nano \
  vim \
  curl \
  ca-certificates \
  wget \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Node.js 20 (LTS) from NodeSource
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
  && apt-get install -y nodejs \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Create the vscode user and grant passwordless sudo.
# The base image ships a default `ubuntu` user at UID/GID 1000; remove it first
# so the vscode user can take those IDs.
RUN userdel -r ubuntu 2>/dev/null || true \
  && groupdel ubuntu 2>/dev/null || true \
  && groupadd --gid $USER_GID $USERNAME \
  && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
  && echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
  && chmod 0440 /etc/sudoers.d/$USERNAME

# Ensure the vscode user has access to /usr/local/share
RUN mkdir -p /usr/local/share/npm-global && \
  chown -R $USERNAME:$USERNAME /usr/local/share

# Persist bash history.
RUN SNIPPET="export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  && mkdir /commandhistory \
  && touch /commandhistory/.bash_history \
  && chown -R $USERNAME /commandhistory

# Set `DEVCONTAINER` environment variable to help with orientation
ENV DEVCONTAINER=true

# Create workspace and config directories and set permissions
RUN mkdir -p /workspace /home/$USERNAME/.claude && \
  chown -R $USERNAME:$USERNAME /workspace /home/$USERNAME/.claude

# Pre-configure Claude Code for sandbox mode inside Docker.
# enableWeakerNestedSandbox is required for bubblewrap to work inside a container.
# The iptables firewall (init-firewall.sh) provides the outer network isolation
# that justifies using the weaker nested sandbox mode.
# Users with a dotfiles manager (chezmoi, etc.) can override this via VS Code's
# dotfiles support, which runs after container creation.
RUN printf '{\n  "sandbox": {\n    "enabled": true,\n    "enableWeakerNestedSandbox": true\n  }\n}\n' \
  > /home/$USERNAME/.claude/settings.json && \
  chown $USERNAME:$USERNAME /home/$USERNAME/.claude/settings.json

WORKDIR /workspace

ARG GIT_DELTA_VERSION=0.18.2
RUN ARCH=$(dpkg --print-architecture) && \
  wget "https://github.com/dandavison/delta/releases/download/${GIT_DELTA_VERSION}/git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  sudo dpkg -i "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  rm "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb"

# Set up non-root user
USER $USERNAME

# Install global packages
ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global
ENV PATH=$PATH:/usr/local/share/npm-global/bin

# Set the default shell to zsh rather than sh
ENV SHELL=/bin/zsh

# Set the default editor and visual
ENV EDITOR=nano
ENV VISUAL=nano

# Default powerline10k theme
ARG ZSH_IN_DOCKER_VERSION=1.2.0
RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v${ZSH_IN_DOCKER_VERSION}/zsh-in-docker.sh)" -- \
  -p git \
  -p fzf \
  -t robbyrussell \
  -a "source /usr/share/doc/fzf/examples/key-bindings.zsh" \
  -a "source /usr/share/doc/fzf/examples/completion.zsh" \
  -a "export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  -x

# Install Claude and sandbox-runtime (for seccomp filters)
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} @anthropic-ai/sandbox-runtime

# Install UV — manages Python toolchain (interpreter versions, project dependencies, virtualenvs)
ARG UV_VERSION=0.10.8
RUN curl -LsSf https://astral.sh/uv/${UV_VERSION}/install.sh | sh
ENV PATH="/home/$USERNAME/.local/bin:$PATH"

# Install chezmoi — dotfiles manager, available on PATH for VS Code dotfiles hook
RUN sh -c "$(curl -fsLS get.chezmoi.io)" -- -b /home/$USERNAME/.local/bin

# Copy and set up firewall script
COPY init-firewall.sh /usr/local/bin/
USER root
RUN chmod +x /usr/local/bin/init-firewall.sh && \
  echo "$USERNAME ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh" > /etc/sudoers.d/$USERNAME-firewall && \
  chmod 0440 /etc/sudoers.d/$USERNAME-firewall
USER $USERNAME
