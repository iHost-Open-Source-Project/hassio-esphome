ARG BUILD_VERSION=dev
ARG BUILD_OS=alpine
ARG BUILD_BASE_VERSION=2025.04.0
ARG BUILD_TYPE=docker

# https://github.com/hassio-addons/addon-debian-base/releases
FROM ghcr.io/hassio-addons/debian-base:7.2.0 AS base-ha-addon
# https://hub.docker.com/_/debian?tab=tags&page=1&name=bookworm
FROM debian:12.2-slim AS base-docker

ARG BUILD_TYPE
FROM base-${BUILD_TYPE} AS base

ARG TARGETARCH
ARG TARGETVARIANT

# Note that --break-system-packages is used below because
# https://peps.python.org/pep-0668/ added a safety check that prevents
# installing packages with the same name as a system package. This is
# not a problem for us because we are not concerned about overwriting
# system packages because we are running in an isolated container.

RUN \
    apt-get update \
    # Use pinned versions so that we get updates with build caching
    && apt-get install -y --no-install-recommends \
        python3-pip \
        python3-setuptools \
        python3-venv \
        python3-wheel \
        iputils-ping \
        git \
        curl \
        openssh-client \
        python3-cffi \
        libcairo2 \
        libmagic1 \
        patch \
        iputils-ping \
        nginx-light \
        build-essential \
        python3-dev \
        zlib1g-dev \
        libjpeg-dev \
        libfreetype-dev \
        libssl-dev \
        libffi-dev \
        cargo \
        pkg-config \
        libtiff6 \
        libopenjp2-7 \
    && rm -rf \
        /tmp/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/*

ENV \
  # Fix click python3 lang warning https://click.palletsprojects.com/en/7.x/python3/
  LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  # Store globally installed pio libs in /piolibs
  PLATFORMIO_GLOBALLIB_DIR=/piolibs

# Support legacy binaries on Debian multiarch system. There is no "correct" way
# to do this, other than using properly built toolchains...
# See: https://unix.stackexchange.com/questions/553743/correct-way-to-add-lib-ld-linux-so-3-in-debian
RUN \
    if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
        ln -s /lib/arm-linux-gnueabihf/ld-linux-armhf.so.3 /lib/ld-linux.so.3; \
    fi

RUN git config --system --add safe.directory "*"

ENV PIP_DISABLE_PIP_VERSION_CHECK=1

COPY requirements.txt /

RUN \
    # Ubuntu python3-pip is missing wheel
    if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
        export PIP_EXTRA_INDEX_URL="https://www.piwheels.org/simple"; \
    fi; \
    pip3 install --break-system-packages --no-cache-dir -r /requirements.txt

RUN \
    platformio settings set enable_telemetry No \
    && platformio settings set check_platformio_interval 1000000 \
    && mkdir -p /piolibs

COPY script/platformio_install_deps.py platformio.ini /
RUN /platformio_install_deps.py /platformio.ini --libraries

ARG BUILD_VERSION

LABEL \
    org.opencontainers.image.authors="The ESPHome Authors" \
    org.opencontainers.image.title="ESPHome" \
    org.opencontainers.image.description="ESPHome is a system to configure your microcontrollers by simple yet powerful configuration files and control them remotely through Home Automation systems" \
    org.opencontainers.image.url="https://esphome.io/" \
    org.opencontainers.image.documentation="https://esphome.io/" \
    org.opencontainers.image.source="https://github.com/esphome/esphome" \
    org.opencontainers.image.licenses="ESPHome" \
    org.opencontainers.image.version=${BUILD_VERSION}


# ======================= docker-type image =======================
#FROM base AS base-docker
#
## Expose the dashboard to Docker
#EXPOSE 6052
#
## Run healthcheck (heartbeat)
#HEALTHCHECK --interval=30s --timeout=30s \
#    CMD curl --fail http://localhost:6052/version -A "HealthCheck" || exit 1
#
#COPY docker/docker_entrypoint.sh /entrypoint.sh
#
## The directory the user should mount their configuration files to
#VOLUME /config
#WORKDIR /config
## Set entrypoint to esphome (via a script) so that the user doesn't have to type 'esphome'
## in every docker command twice
#ENTRYPOINT ["/entrypoint.sh"]
## When no arguments given, start the dashboard in the workdir
#CMD ["dashboard", "/config"]


# ======================= ha-addon-type image =======================
FROM base AS hassio

# Copy root filesystem
COPY docker/ha-addon-rootfs/ /

# Copy esphome and install
COPY . /esphome
RUN pip3 install --break-system-packages --no-cache-dir -e /esphome

ARG BUILD_VERSION
LABEL \
    io.hass.name="ESPHome" \
    io.hass.description="ESPHome is a system to configure your microcontrollers by simple yet powerful configuration files and control them remotely through Home Automation systems" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}"
    # io.hass.arch is inherited from addon-debian-base
