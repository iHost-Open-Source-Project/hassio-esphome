ARG BUILD_VERSION=dev
ARG BUILD_OS=alpine
ARG BUILD_BASE_VERSION=2025.04.0
ARG BUILD_TYPE=docker

# FROM ghcr.io/ihost-open-source-project/esphome-docker-base:${BUILD_OS}-${BUILD_BASE_VERSION} AS base-source-docker
FROM ghcr.io/ihost-open-source-project/esphome-docker-base:${BUILD_OS}-ha-addon-${BUILD_BASE_VERSION} AS base-source-ha-addon

ARG BUILD_TYPE
FROM base-source-${BUILD_TYPE} AS base

RUN git config --system --add safe.directory "*"

ENV PIP_DISABLE_PIP_VERSION_CHECK=1

RUN pip install --no-cache-dir -U pip uv==0.6.14

# 新增：设置 piwheels 镜像用于加速 armv7 架构的 pip 安装
RUN if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
      export PIP_EXTRA_INDEX_URL="https://www.piwheels.org/simple"; \
    fi && \
    pip3 install --break-system-packages --no-cache-dir platformio==6.1.18


COPY requirements.txt /

# RUN \
#     uv pip install --no-cache-dir \
#     -r /requirements.txt

# 安装所有构建依赖，确保兼容 armv7/arm64/amd64 编译流程
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Python 构建链
    python3-pip \
    python3-setuptools \
    python3-venv \
    python3-wheel \
    python3-dev \
    python3-cffi \
    # C/C++ 编译工具链
    gcc \
    g++ \
    make \
    build-essential \
    libffi-dev \
    libssl-dev \
    patch \
    pkg-config \
    # Rust 构建链
    cargo \
    # 图形/图像依赖（Pillow 用于图标渲染）
    libjpeg-dev \
    zlib1g-dev \
    libfreetype-dev \
    libtiff-dev \
    libopenjp2-7 \
    libcairo2 \
    libmagic1 \
    # 网络与调试工具
    iputils-ping \
    curl \
    git \
    openssh-client \
    && rm -rf /var/lib/apt/lists/*

# 新增：设置 piwheels 镜像用于加速 armv7 架构的 pip 安装
RUN if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
      export PIP_EXTRA_INDEX_URL="https://www.piwheels.org/simple"; \
    fi && \
    pip3 install --break-system-packages --no-cache-dir platformio==6.1.18

RUN \
    platformio settings set enable_telemetry No \
    && platformio settings set check_platformio_interval 1000000 \
    && mkdir -p /piolibs 


COPY script/platformio_install_deps.py platformio.ini /
RUN /platformio_install_deps.py /platformio.ini --libraries

# 安装 Python 依赖（优先使用 piwheels）
RUN if [ "$TARGETARCH$TARGETVARIANT" = "armv7" ]; then \
      export PIP_EXTRA_INDEX_URL="https://www.piwheels.org/simple"; \
    fi && \
    pip3 install --break-system-packages --no-cache-dir -r /requirements.txt -r /requirements_optional.txt

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
FROM base AS base-docker

# Expose the dashboard to Docker
EXPOSE 6052

# Run healthcheck (heartbeat)
HEALTHCHECK --interval=30s --timeout=30s \
    CMD curl --fail http://localhost:6052/version -A "HealthCheck" || exit 1

COPY docker/docker_entrypoint.sh /entrypoint.sh

# The directory the user should mount their configuration files to
VOLUME /config
WORKDIR /config
# Set entrypoint to esphome (via a script) so that the user doesn't have to type 'esphome'
# in every docker command twice
ENTRYPOINT ["/entrypoint.sh"]
# When no arguments given, start the dashboard in the workdir
CMD ["dashboard", "/config"]


# ======================= ha-addon-type image =======================
FROM base AS base-ha-addon

# Copy root filesystem
COPY docker/ha-addon-rootfs/ /

ARG BUILD_VERSION
LABEL \
    io.hass.name="ESPHome" \
    io.hass.description="ESPHome is a system to configure your microcontrollers by simple yet powerful configuration files and control them remotely through Home Automation systems" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}"
    # io.hass.arch is inherited from addon-debian-base

ARG BUILD_TYPE
FROM base-${BUILD_TYPE} AS final

# Copy esphome and install
COPY . /esphome
RUN uv pip install --no-cache-dir -e /esphome
