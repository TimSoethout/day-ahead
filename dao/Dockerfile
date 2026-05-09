ARG BUILD_FROM
FROM ${BUILD_FROM} AS builder

ARG BUILD_ARCH
ENV VIRTUAL_ENV=/root/dao/venv/day_ahead
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN if command -v apt-get >/dev/null 2>&1; then \
    apt-get update -y && \
    apt-get install -y --no-install-recommends \
      gcc g++ pkgconf python3 python3-dev python3-pip python3-venv && \
    apt-get clean && rm -rf /var/lib/apt/lists/*; \
  elif command -v apk >/dev/null 2>&1; then \
    apk add --no-cache \
      build-base pkgconf python3 python3-dev py3-pip py3-virtualenv; \
  else \
    echo "Unsupported package manager in base image" >&2; \
    exit 1; \
  fi

COPY requirements.txt /tmp/requirements.txt
RUN python3 -m venv $VIRTUAL_ENV && \
  python -m pip install --no-cache-dir -r /tmp/requirements.txt && \
  rm -rf /root/.cache /var/tmp/* /tmp/* && \
  # XGBoost pulls optional CUDA/NCCL wheel artifacts; DAO runs CPU-only in HA.
  find "$VIRTUAL_ENV/lib/python"*/site-packages -maxdepth 1 -type d \( -name 'nvidia' -o -name 'nvidia_*' -o -name 'nvidia-*' \) -exec rm -rf {} + && \
  find "$VIRTUAL_ENV/lib/python"*/site-packages -type d \( -name tests -o -name test -o -name docs -o -name doc -o -name examples -o -name example \) -prune -exec rm -rf {} + && \
  # MIP uses cbcbox; keep generic/avx2 solver binaries and drop debug-only build.
  rm -rf "$VIRTUAL_ENV/lib/python"*/site-packages/cbcbox/cbc_dist_debug* && \
  find "$VIRTUAL_ENV/lib" -type f -name '*.a' -delete && \
  find "$VIRTUAL_ENV/lib" -type d -name __pycache__ -prune -exec rm -rf {} + && \
  find "$VIRTUAL_ENV/lib" -type f \( -name '*.pyc' -o -name '*.pyo' \) -delete

FROM ${BUILD_FROM}

ARG BUILD_ARCH
ENV PYTHONPATH="/root:/root/dao:/root/dao/prog"
ENV VIRTUAL_ENV=/root/dao/venv/day_ahead
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN if command -v apt-get >/dev/null 2>&1; then \
    apt-get update -y && \
    # DB drivers are Python wheels; only runtime python + inotify are required.
    apt-get install -y --no-install-recommends inotify-tools python3 && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /root/.cache; \
  elif command -v apk >/dev/null 2>&1; then \
    # DB drivers are Python wheels; only runtime python + inotify are required.
    apk add --no-cache inotify-tools python3 && \
    rm -rf /tmp/* /var/tmp/* /root/.cache; \
  else \
    echo "Unsupported package manager in base image" >&2; \
    exit 1; \
  fi

# Copy software for add-on
COPY __init__.py /root/dao/
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
