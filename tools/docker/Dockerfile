# syntax=docker/dockerfile:1.11.0@sha256:1f2be5a2aa052cbd9aedf893d17c63277c3d1c51b3fb0f3b029c6b34f658d057

ARG BASE_IMAGE_TYPE=slim

# --------------------------------------
# slim image
# --------------------------------------
FROM ghcr.io/renovatebot/base-image:9.3.0@sha256:9266c218c014fbd3501cddef0144810e4e7b2623e2b975d411cdf8b62f97759a AS slim-base

# --------------------------------------
# full image
# --------------------------------------
FROM ghcr.io/renovatebot/base-image:9.3.0-full@sha256:1bb796adb004ca19089e035c217561df177e46e6efa0851b95cc90e4dd896298 AS full-base

ENV RENOVATE_BINARY_SOURCE=global

# --------------------------------------
# build image
# --------------------------------------
FROM --platform=$BUILDPLATFORM ghcr.io/renovatebot/base-image:9.3.0@sha256:9266c218c014fbd3501cddef0144810e4e7b2623e2b975d411cdf8b62f97759a AS build

# We want a specific node version here
# renovate: datasource=node-version
RUN install-tool node 22.11.0

WORKDIR /usr/local/renovate

ARG TARGETPLATFORM
ARG TARGETARCH
ARG BUILDPLATFORM

RUN set -ex; \
  echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM"; \
  uname -a; \
  true

# replace `amd64` with `x86_64` for `node`
ENV ARCH=${TARGETARCH/amd64/x86_64}
ENV ARCH=${ARCH/arm64/aarch64}

# fetch static node binary
RUN set -ex; \
  ver=$(node --version); ver=${ver:1} \
  temp_dir="$(mktemp -d)"; \
  curl -fsSL "https://github.com/containerbase/node-prebuild/releases/download/${ver}/node-${ver}-${ARCH}.tar.xz" -o ${temp_dir}/node.tar.xz; \
  bsdtar --strip 1 -C ${temp_dir} -xf ${temp_dir}/node.tar.xz; \
  cp ${temp_dir}/bin/node ./node; \
  true

# fetch npm packages
ENV CI=1 npm_config_modules_cache_max_age=0 \
  npm_config_loglevel=info


# replace `amd64` with `x64` for `node`
ENV ARCH=${TARGETARCH/amd64/x64}

COPY --link pnpm-lock.yaml ./

# set `npm_config_arch` for `prebuild-install`
# set `npm_config_platform_arch` for `install-artifact-from-github`
# only fetch deps from lockfile https://pnpm.io/cli/fetch
RUN set -ex; \
  export npm_config_arch=${ARCH} npm_config_platform_arch=${ARCH}; \
  corepack pnpm fetch --prod; \
  true

COPY --link . ./

# install npm packages
RUN set -ex; \
  corepack pnpm install --prod --offline --ignore-scripts; \
  true

# --------------------------------------
# final image
# --------------------------------------
FROM ${BASE_IMAGE_TYPE}-base

LABEL name="renovate"
LABEL org.opencontainers.image.source="https://github.com/renovatebot/renovate" \
  org.opencontainers.image.url="https://renovatebot.com" \
  org.opencontainers.image.licenses="AGPL-3.0-only"

WORKDIR /usr/src/app

COPY tools/docker/bin/ /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/renovate-entrypoint.sh"]
CMD ["renovate"]

ARG RENOVATE_VERSION

COPY --link --from=build --chown=root:root /usr/local/renovate/ /usr/local/renovate/

# test
RUN set -ex; \
  renovate --version; \
  pushd /usr/local/renovate/; \
  ./node -e "new require('re2')('.*').exec('test');new require('better-sqlite3')(':memory:')"; \
  true

LABEL \
  org.opencontainers.image.version="${RENOVATE_VERSION}" \
  org.label-schema.version="${RENOVATE_VERSION}"

# Numeric user ID for the ubuntu user. Used to indicate a non-root user to OpenShift
USER 12021
