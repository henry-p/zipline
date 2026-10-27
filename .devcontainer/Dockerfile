# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM debian:trixie-slim AS python
ARG TARGETARCH
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl && rm -rf /var/lib/apt/lists/*
RUN case "$TARGETARCH" in \
      amd64) python_archive_url="https://github.com/astral-sh/python-build-standalone/releases/download/20261026/cpython-3.13.18%2B20261026-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz"; python_archive_sha="df15fe60d3ff55bf9dcf25824b5572756bb8816ff93865a273d3301d7066a8b7" ;; \
      arm64) python_archive_url="https://github.com/astral-sh/python-build-standalone/releases/download/20261026/cpython-3.13.18%2B20261026-aarch64-unknown-linux-gnu-install_only_stripped.tar.gz"; python_archive_sha="578d9da1983463ffb0c8c98ab0da2f18177f44069328eb0af847860c1eb05c3e" ;; \
      *) exit 1 ;; \
    esac \
 && curl --fail --location --retry 3 "$python_archive_url" -o /tmp/python.tar.gz \
 && printf '%s  /tmp/python.tar.gz\n' "$python_archive_sha" | sha256sum -c - \
 && mkdir /python && tar -xzf /tmp/python.tar.gz -C /python --strip-components=1

FROM debian:trixie-slim AS development
ENV PATH=/usr/local/bin:$PATH LANG=C.UTF-8 SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt CC=gcc CXX=g++
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates git build-essential gfortran libfreetype6-dev libpng-dev libopenblas-dev liblapack-dev libhdf5-dev \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --create-home --uid 1000 --shell /bin/bash vscode
COPY --from=python /python /usr/local
COPY requirements-security.txt /tmp/requirements-security.txt
RUN python3 -c "import sys; assert sys.version_info[:3] == (3, 13, 18)" \
 && python3 -m pip install --no-cache-dir --upgrade pip==26.2.2 certifi==2026.10.23 \
 && python3 -m pip install --no-cache-dir -r /tmp/requirements-security.txt \
 && python3 -m pip check
USER vscode


FROM development AS validation
USER root
WORKDIR /src
COPY . ./
ENV SETUPTOOLS_SCM_PRETEND_VERSION=3.2.1
RUN python3 -m pip install --no-cache-dir --no-deps . \
    && mkdir /evidence && python3 -m pip freeze > /evidence/installed.txt
RUN --network=none python3 -m pytest -q tests/finance tests/test_algorithm.py tests/test_assets.py tests/utils/test_calendar_utils.py --junitxml=/evidence/tests.xml > /evidence/tests.log 2>&1 || { tail -n 120 /evidence/tests.log; exit 1; }
RUN tail -n 5 /evidence/tests.log

FROM scratch AS evidence
COPY --from=validation /evidence/ /evidence/
