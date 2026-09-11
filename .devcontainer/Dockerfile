FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

RUN apt-get update && \
    apt-get install -y \
        curl \
        ca-certificates \
        usbutils \
        && rm -rf /var/lib/apt/lists/*

# Install Arduino CLI
RUN curl -fsSL \
    https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh \
    | BINDIR=/usr/local/bin sh

# Configure Arduino CLI and install Arduino AVR core
RUN arduino-cli config init && \
    arduino-cli core update-index && \
    arduino-cli core install arduino:avr
