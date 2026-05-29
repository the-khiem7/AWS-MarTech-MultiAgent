FROM mcr.microsoft.com/devcontainers/base:ubuntu

ARG HUGO_VERSION=0.134.3
ENV DIAGRAMS_VENV=/opt/diagrams-venv
ENV PATH="${DIAGRAMS_VENV}/bin:${PATH}"

COPY .devcontainer/requirements-diagrams.txt /tmp/requirements-diagrams.txt

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      fd-find \
      graphviz \
      imagemagick \
      jq \
      libwebp-dev \
      libwebp7 \
      pngquant \
      tesseract-ocr \
      webp \
      optipng \
      python3 \
      python3-pip \
      python3-venv \
      tar; \
    python3 -m venv "${DIAGRAMS_VENV}"; \
    "${DIAGRAMS_VENV}/bin/pip" install --no-cache-dir -r /tmp/requirements-diagrams.txt; \
    ARCH="$(dpkg --print-architecture)"; \
    case "${ARCH}" in \
      amd64) HUGO_ARCH='amd64' ;; \
      arm64) HUGO_ARCH='arm64' ;; \
      *) echo "Unsupported architecture: ${ARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-${HUGO_ARCH}.tar.gz" \
      | tar -xz -C /usr/local/bin hugo; \
    chown -R vscode:vscode "${DIAGRAMS_VENV}"; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /tmp/requirements-diagrams.txt
