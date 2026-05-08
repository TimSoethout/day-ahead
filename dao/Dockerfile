ARG BUILD_FROM
FROM ${BUILD_FROM}

ARG BUILD_ARCH
ENV BUILD_ARCH=$BUILD_ARCH
ENV PYTHONPATH="/root:/root/dao:/root/dao/prog"
ENV VIRTUAL_ENV=/root/dao/venv/day_ahead
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
  apt-get update -y && \
  apt-get install -y --no-install-recommends pkgconf mariadb-client inotify-tools procps \
    gunicorn libmariadb3 libmariadb-dev postgresql-client sqlite3 libjpeg-dev libffi-dev \
    wget libnauty-dev coinor-libcgl-dev gcc g++ git \
    python3 python3-pip python3-venv && \
  rm -rf /var/lib/apt/lists/*

# Copy software for add-on
RUN mkdir /root/dao
COPY __init__.py /root/dao
COPY lib /root/dao/lib/
COPY prog /root/dao/prog/
COPY webserver /root/dao/webserver/
COPY data /tmp/daodata/

#version
ARG BUILD_VERSION
ENV DAO_VERSION=$BUILD_VERSION
RUN printf  '__version__ ="%s"\n' "$DAO_VERSION" > /root/dao/prog/version.py


WORKDIR /root/dao/prog
RUN python3 -m venv $VIRTUAL_ENV
RUN --mount=type=bind,target=/tmp \
  pip install --no-cache-dir uv && \
  if [ "x${BUILD_ARCH}" = "xaarch64" ]; then ln -sf "$(command -v mariadb_config)" /usr/local/bin/mariadb_config; fi && \
  TMPDIR=/var/tmp UV_NO_CACHE=1 uv pip install -r /tmp/requirements.txt && \
  pip uninstall -y uv

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
