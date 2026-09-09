## Copyright (C) 2025 - 2025 ENCRYPTED SUPPORT LLC <adrelanos@whonix.org>
## See the file COPYING for copying conditions.

## debian:trixie-slim manifest digest as of 2026-05-08, resolved via
## the Docker Hub registry API. Re-resolve with:
##   curl -sS -H "Authorization: Bearer $(curl -s 'https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/debian:pull' | jq -r .token)" \
##        -H 'Accept: application/vnd.docker.distribution.manifest.list.v2+json' \
##        -D - -o /dev/null \
##        https://registry-1.docker.io/v2/library/debian/manifests/trixie-slim \
##        | grep -i docker-content-digest
FROM debian:trixie-slim@sha256:cedb1ef40439206b673ee8b33a46a03a0c9fa90bf3732f54704f99cb061d2c5a AS baseimage

ENV \
USER=user \
HOME=/home/user

## Create the in-container 'user' with this uid. Defaults to 1000 for local
## use; CI passes --build-arg DM_UID=$(id -u) so the container user matches
## the host runner (uid 1001 on GitHub runners), required for the bind-mounted
## volumes to be writable during the build and readable by the runner after.
ARG DM_UID=1000
ENV DM_UID=${DM_UID}

## Install ca-certificates and ensure the expected files from it are present.
COPY docker/apt-bootstrap-ca-certificates /usr/bin
RUN chmod 0755 /usr/bin/apt-bootstrap-ca-certificates \
    && /usr/bin/apt-bootstrap-ca-certificates

## Provide the container a timestamp-pinned sources file. We use apt options to
## point apt at this file, so it doesn't need to be in sources.list.d.
COPY build_sources/debian_stable_frozen_direct_clearnet.sources.in /etc/apt/dm-frozen.sources

## Fix any umask-induced permission issues.
RUN chmod 0644 /etc/apt/dm-frozen.sources

## Populate the timestamp in the frozen sources file. Note that
## apt-populate-frozen-timestamp deletes /etc/apt/frozen-snapshot-timestamp
## when it's done with it.
COPY docker/apt-populate-frozen-timestamp /usr/bin
COPY build_sources/frozen-snapshot-timestamp /etc/apt/frozen-snapshot-timestamp
RUN chmod 0775 /usr/bin/apt-populate-frozen-timestamp \
    && /usr/bin/apt-populate-frozen-timestamp

COPY docker/derivative-maker-docker-setup /usr/bin
COPY docker/build-data/docker-entrypoint.target /etc/systemd/system
COPY docker/build-data/docker-entrypoint.service /etc/systemd/system
COPY docker/build-data/docker-entrypoint-stop.sh /usr/bin

RUN /usr/bin/derivative-maker-docker-setup

FROM baseimage

LABEL org.opencontainers.image.authors="adrelanos@whonix.org"
LABEL org.opencontainers.image.description="Containerization of Kicksecure/derivative-maker"
LABEL org.opencontainers.image.title="derivative-maker-docker"
LABEL org.opencontainers.image.source="https://github.com/Kicksecure/derivative-maker"
LABEL org.opencontainers.image.documentation="https://www.kicksecure.com/wiki/Dev/Build_Documentation/VM"
LABEL org.opencontainers.image.version="1.0"
LABEL org.opencontainers.image.vendor="ENCRYPTED SUPPORT LLC"

COPY docker/entrypoint.sh /usr/bin
COPY docker/derivative-maker-docker-start /usr/bin

## Fix any umask-induced permission issues.
RUN chmod 0755 /usr/bin/entrypoint.sh /usr/bin/derivative-maker-docker-start

ENTRYPOINT ["/usr/bin/entrypoint.sh"]

CMD ["/bin/bash"]
