ARG ZOLA_VERSION=0.19.2

FROM ghcr.io/getzola/zola:v${ZOLA_VERSION} AS zola

FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

COPY --from=zola /bin/zola /usr/local/bin/zola

RUN zola --version
