# containers/code-server/Dockerfile
#
# OOD code-server image for NCAR.
#
# Inclusion rule: if a user cannot install it into their own home without root,
# it belongs here. Anything a user can drop into ~/.local/bin themselves is
# deliberately left out -- including kubectl and kubelogin.
#
# This image runs under Open OnDemand, which maps the pod to the real user's
# UID through a mounted /etc/passwd. Nothing here is root at runtime and users
# cannot apt-install, which is what makes the inclusion rule load-bearing
# rather than a style preference.
#
# Build (CI does this on every push touching this directory -- see
# .github/workflows/build-code-server.yml):
#   docker build -t hub.k8s.ucar.edu/cirrus-ood/code-server:<tag> containers/code-server

FROM docker.io/codercom/code-server:4.131.0

USER root
# Note: podman/buildah silently ignore SHELL when emitting OCI-format images
# ("SHELL is not supported for OCI image format"). Every RUN below therefore
# sets its own `set -eux` and avoids pipes rather than relying on pipefail.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Verified against upstream on 2026-09-15; pinned so a rebuild is reproducible
# rather than drifting with whatever is newest.
ARG MINIFORGE_VERSION=26.7.2-0
ARG BUILDKIT_VERSION=0.33.0

# --- Toolchain (root required) ----------------------------------------------
# Compilers and headers for the C++/Fortran users. clangd is the C/C++ language
# server: ms-vscode.cpptools is not on Open VSX, and code-server cannot use the
# Microsoft marketplace, so clangd is the only workable intellisense path here.
#
# curl, unzip, tar and xz-utils are load-bearing beyond their own use: they are
# what make the "user installs it themselves" path actually work. Without unzip
# a user cannot unpack kubelogin at all.
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      build-essential gfortran cmake ninja-build pkg-config gdb \
      clangd clang-format \
      python3 python3-pip python3-venv \
      git git-lfs openssh-client \
      curl wget ca-certificates unzip zip xz-utils \
      jq ripgrep tmux vim nano less rsync procps \
      locales tzdata; \
    rm -rf /var/lib/apt/lists/*

# Generate en_US.UTF-8 so LANG can actually be set. Without this a bashrc that
# validates the locale before setting it correctly refuses on a bare Debian
# image, and everything runs in POSIX collation.
RUN set -eux; \
    sed -i 's/^# *\(en_US.UTF-8 UTF-8\)/\1/' /etc/locale.gen; \
    locale-gen
ENV LANG=en_US.UTF-8 \
    LC_ALL=

# --- ~/.local/bin on PATH for everyone --------------------------------------
# This is what makes the self-install path viable for users who do not bring
# their own dotfiles. code-server terminals are non-login interactive shells,
# so ~/.profile never runs, and Debian's default ~/.bashrc does not add
# ~/.local/bin either -- /etc/bash.bashrc is the only global hook that fires.
# Idempotent, so it composes with dotfiles that prepend the same dir.
#
# CRITICAL: deliberately NOT sourcing conda here. conda.sh exports CONDA_SHLVL,
# and /etc/bash.bashrc runs before ~/.bashrc -- that would silently suppress
# dotfiles that source a user's own ~/miniforge3 behind a CONDA_SHLVL guard.
# Written with printf rather than a shell heredoc on purpose: heredocs need the
# BuildKit Dockerfile frontend, and podman/buildah's imagebuilder -- the likely
# builder on an HPC host -- neither supports them nor honors a `# syntax`
# directive. It ends the RUN at the `cat` line and then tries to parse `if` as a
# Dockerfile instruction. printf builds anywhere.
RUN set -eux; \
    printf '%s\n' \
      '' \
      '# Per-user binaries (kubectl, kubelogin, krew, pipx, ...) live here.' \
      'if [ -n "${HOME:-}" ]; then' \
      '    case ":$PATH:" in' \
      '        *":$HOME/.local/bin:"*) ;;' \
      '        *) PATH="$HOME/.local/bin:$PATH" ;;' \
      '    esac' \
      'fi' \
      >> /etc/bash.bashrc

# --- Miniforge (conda-forge) at /opt/conda ----------------------------------
# Strictly speaking this breaks the inclusion rule -- conda installs into $HOME
# without root. It is here anyway because the alternative is every user pulling
# ~500MB+ into their GLADE home and solving environments over NFS. Users who
# want their own ~/miniforge3 are unaffected.
#
# Miniforge rather than Anaconda: the Anaconda ToS requires a paid license for
# organizations over 200 people using the defaults channel, and we are well
# over that.
#
# Miniforge names its release assets x86_64/aarch64 (uname -m), NOT the
# amd64/arm64 that buildkit and kubectl use. Do not reuse
# `dpkg --print-architecture` for this one.
#
# Only condabin goes on PATH -- it holds the conda entry point and nothing
# else, so the system python3 is not shadowed for users who do not want conda.
# Base is not activated and conda init is not run; users opt in.
#
# /opt/conda stays root-owned. conda notices the envs dir is unwritable and
# falls back to ~/.conda/envs (and ~/.conda/pkgs) on its own. That is intended:
# the shared base is the part worth not copying per user.
RUN set -eux; \
    arch="$(uname -m)"; \
    curl -fsSLo /tmp/miniforge.sh \
      "https://github.com/conda-forge/miniforge/releases/download/${MINIFORGE_VERSION}/Miniforge3-${MINIFORGE_VERSION}-Linux-${arch}.sh"; \
    bash /tmp/miniforge.sh -b -p /opt/conda; \
    rm -f /tmp/miniforge.sh; \
    /opt/conda/bin/conda clean -afy; \
    for f in conda mamba; do \
      if [ -r "/opt/conda/etc/profile.d/${f}.sh" ]; then \
        ln -sf "/opt/conda/etc/profile.d/${f}.sh" "/etc/profile.d/${f}.sh"; \
      fi; \
    done
ENV PATH=/opt/conda/condabin:${PATH}

# --- buildctl ---------------------------------------------------------------
# Client only, no daemon -- it talks to our remote buildkitd. Also a rule
# exception (a user could unpack this tarball themselves); included because
# remote builds are a first-class workflow here and the binary is small.
#
# BUILDKIT_HOST is set in the OOD submit.yml.erb, not here, so the image
# carries no site topology.
RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    curl -fsSLo /tmp/buildkit.tar.gz \
      "https://github.com/moby/buildkit/releases/download/v${BUILDKIT_VERSION}/buildkit-v${BUILDKIT_VERSION}.linux-${arch}.tar.gz"; \
    tar -xzf /tmp/buildkit.tar.gz -C /usr/local/bin --strip-components=1 bin/buildctl; \
    rm -f /tmp/buildkit.tar.gz

USER 1000
