# syntax=docker/dockerfile:1

# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfiles.

# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/
####################### VAULT BUILD IMAGE  #######################
# The web-vault digest specifies a particular web-vault build on Docker Hub.
# Using the digest instead of the tag name provides better security,
# as the digest of an image is immutable, whereas a tag name can later
# be changed to point to a malicious image.
#
# To verify the current digest for a given tag name:
# - From https://hub.docker.com/r/vaultwarden/web-vault/tags,
#   click the tag name to view the digest of the image it currently points to.
# - From the command line:
#     $ docker pull vaultwarden/web-vault:v2.27.0
#     $ docker image inspect --format "{{.RepoDigests}}" vaultwarden/web-vault:v2.27.0
#     [vaultwarden/web-vault@sha256:e0cb877daa59677b0031310ee23c00a9ee2806840b40f2319867002bf15ff9aa]
#
# - Conversely, to get the tag name from the digest:
#     $ docker image inspect --format "{{.RepoTags}}" vaultwarden/web-vault@sha256:e0cb877daa59677b0031310ee23c00a9ee2806840b40f2319867002bf15ff9aa
#     [vaultwarden/web-vault:v2.27.0]
#
FROM vaultwarden/web-vault@sha256:e0cb877daa59677b0031310ee23c00a9ee2806840b40f2319867002bf15ff9aa as vault

########################## BUILD IMAGE  ##########################
FROM rust:1.59-bullseye as build



# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    TZ=UTC \
    TERM=xterm-256color \
    CARGO_HOME="/root/.cargo" \
    USER="root"


# Create CARGO_HOME folder and don't download rust docs
RUN mkdir -pv "${CARGO_HOME}" \
    && rustup set profile minimal

#
# Install required build libs for armhf architecture.
# hadolint ignore=DL3059
RUN dpkg --add-architecture armhf \
    && apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        libssl-dev:armhf \
        libc6-dev:armhf \
        libpq5:armhf \
        libpq-dev:armhf \
        libmariadb3:armhf \
        libmariadb-dev:armhf \
        libmariadb-dev-compat:armhf \
        gcc-arm-linux-gnueabihf \
    #
    # Make sure cargo has the right target config
    && echo '[target.armv7-unknown-linux-gnueabihf]' >> "${CARGO_HOME}/config" \
    && echo 'linker = "arm-linux-gnueabihf-gcc"' >> "${CARGO_HOME}/config" \
    && echo 'rustflags = ["-L/usr/lib/arm-linux-gnueabihf"]' >> "${CARGO_HOME}/config"

# Set arm specific environment values
ENV CC_armv7_unknown_linux_gnueabihf="/usr/bin/arm-linux-gnueabihf-gcc" \
    CROSS_COMPILE="1" \
    OPENSSL_INCLUDE_DIR="/usr/include/arm-linux-gnueabihf" \
    OPENSSL_LIB_DIR="/usr/lib/arm-linux-gnueabihf"


# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain ./rust-toolchain
COPY ./build.rs ./build.rs

RUN rustup target add armv7-unknown-linux-gnueabihf

# Configure the DB ARG as late as possible to not invalidate the cached layers above
ARG DB=sqlite,mysql,postgresql

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release --target=armv7-unknown-linux-gnueabihf \
    && find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
# hadolint ignore=DL3059
RUN cargo build --features ${DB} --release --target=armv7-unknown-linux-gnueabihf

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM balenalib/armv7hf-debian:bullseye

ENV ROCKET_PROFILE="release" \
    ROCKET_ADDRESS=0.0.0.0 \
    ROCKET_PORT=80

# hadolint ignore=DL3059
RUN [ "cross-build-start" ]

# Create data folder and Install needed libraries
RUN mkdir /data \
    && apt-get update && apt-get install -y \
    --no-install-recommends \
    openssl \
    ca-certificates \
    curl \
    dumb-init \
    libmariadb-dev-compat \
    libpq5 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# hadolint ignore=DL3059
RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY --from=vault /web-vault ./web-vault
COPY --from=build /app/target/armv7-unknown-linux-gnueabihf/release/vaultwarden .

COPY docker/healthcheck.sh /healthcheck.sh
COPY docker/start.sh /start.sh

HEALTHCHECK --interval=60s --timeout=10s CMD ["/healthcheck.sh"]

# Configures the startup!
# We should be able to remove the dumb-init now with Rocket 0.5
# But the balenalib images have some issues with there entry.sh
# See: https://github.com/balena-io-library/base-images/issues/735
# Lets keep using dumb-init for now, since that is working fine.
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/start.sh"]
