FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

USER root

# ----------------------------------------------------------------------
# Base development utilities
# ----------------------------------------------------------------------

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        usbutils \
    && rm -rf /var/lib/apt/lists/*


# ----------------------------------------------------------------------
# Allow the VS Code user to access serial devices
# ----------------------------------------------------------------------

RUN usermod -aG dialout vscode


# ----------------------------------------------------------------------
# Install Arduino CLI
# ----------------------------------------------------------------------

RUN curl -fsSL \
        https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh \
    | BINDIR=/usr/local/bin sh


# ----------------------------------------------------------------------
# Configure Arduino environment
# ----------------------------------------------------------------------

USER vscode

RUN arduino-cli config init \
    && arduino-cli core update-index \
    && arduino-cli core install arduino:avr


# ----------------------------------------------------------------------
# Default shell
# ----------------------------------------------------------------------

SHELL ["/bin/bash", "-c"]