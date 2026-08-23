ARG HERMES_TAG=latest
FROM nousresearch/hermes-agent:${HERMES_TAG}

USER root

# Keep this layer limited to the virtual X11 desktop and its accessibility/VNC
# plumbing. Chromium still runs as the unprivileged upstream "hermes" user.
RUN apt-get -o Acquire::Retries=3 update && \
    apt-get -o Acquire::Retries=3 install -y --no-install-recommends \
        at-spi2-core \
        chromium \
        dbus-x11 \
        openbox \
        x11-utils \
        x11vnc \
        xvfb && \
    rm -rf /var/lib/apt/lists/*

# The official image does not currently contain cua-driver. Install a pinned
# Linux release outside /opt/data so the named volume cannot hide it. Rebuild
# with --build-arg CUA_DRIVER_VERSION=... to update it deliberately.
ARG CUA_DRIVER_VERSION=0.21.0
RUN curl -fsSL --retry 3 \
        https://raw.githubusercontent.com/trycua/cua/main/libs/cua-driver/scripts/install.sh \
        -o /tmp/install-cua-driver.sh && \
    CUA_DRIVER_RS_VERSION="${CUA_DRIVER_VERSION}" \
    CUA_DRIVER_RS_HOME=/opt/hermes/.cua-driver \
    CUA_DRIVER_RS_INSTALL_DIR=/usr/local/bin \
    CUA_DRIVER_RS_NO_MODIFY_PATH=1 \
        /bin/bash /tmp/install-cua-driver.sh \
            --bin-dir /usr/local/bin --no-modify-path && \
    rm /tmp/install-cua-driver.sh && \
    cua-driver --version

COPY desktop/cont-init.d/10-hermes-desktop /etc/cont-init.d/10-hermes-desktop
COPY desktop/s6-rc.d/ /etc/s6-overlay/s6-rc.d/
