ARG BUILD_FROM
FROM ${BUILD_FROM} AS builder

ARG BUILD_ARCH
ENV VIRTUAL_ENV=/root/dao/venv/day_ahead
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
  apt-get update -y && \
  apt-get install -y --no-install-recommends pkgconf libmariadb-dev gcc g++ \
    python3 python3-dev python3-pip python3-venv

COPY requirements.txt /tmp/requirements.txt
RUN python3 -m venv $VIRTUAL_ENV && \
  pip install --no-cache-dir uv && \
  if [ "x${BUILD_ARCH}" = "xaarch64" ]; then \
    mariadb_cfg="$(command -v mariadb_config)"; \
    [ -n "$mariadb_cfg" ] || { echo "mariadb_config not found" >&2; exit 1; }; \
    ln -s "$mariadb_cfg" /usr/local/bin/mariadb_config; \
  fi && \
  TMPDIR=/var/tmp UV_NO_CACHE=1 uv pip install -r /tmp/requirements.txt && \
  pip uninstall -y uv

FROM ${BUILD_FROM}

ARG BUILD_ARCH
ENV PYTHONPATH="/root:/root/dao:/root/dao/prog"
ENV VIRTUAL_ENV=/root/dao/venv/day_ahead
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
  apt-get update -y && \
  apt-get install -y --no-install-recommends mariadb-client inotify-tools procps \
    libmariadb3 postgresql-client sqlite3 python3 python3-venv

# Copy software for add-on
RUN mkdir /root/dao
COPY __init__.py /root/dao
COPY lib /root/dao/lib/
COPY prog /root/dao/prog/
COPY webserver /root/dao/webserver/
COPY data /tmp/daodata/
COPY --from=builder /root/dao/venv /root/dao/venv

#version
ARG BUILD_VERSION
ENV DAO_VERSION=$BUILD_VERSION
RUN printf  '__version__ ="%s"\n' "$DAO_VERSION" > /root/dao/prog/version.py

WORKDIR /root/dao/prog
COPY run /root/dao/prog/

EXPOSE 5000
WORKDIR /root/dao/prog
RUN chmod a+x run.sh
ENTRYPOINT  ["./run.sh"]

# Labels
LABEL \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION}
