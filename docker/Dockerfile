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

## Trust store first: the pinned sources below are https, and the slim base image
## has no ca-certificates (Priority: standard, so not in the base system). Runs
## against the base image's live sources -- the one unpinned fetch here, explained
## in the script.
COPY apt-bootstrap-ca-certificates /usr/bin
RUN chmod 0755 /usr/bin/apt-bootstrap-ca-certificates \
    && /usr/bin/apt-bootstrap-ca-certificates

## Static apt configuration, pinned to the same snapshot.debian.org timestamp the
## build uses under '--freshness frozen'. Shipped as files rather than generated
## at image-build time, so what apt reads is reviewable in the repo. apt/sources.list
## is empty and replaces the base image's, leaving the snapshot as the only source.
COPY apt/debian.sources /etc/apt/sources.list.d/debian.sources
COPY apt/99dm-frozen-snapshot /etc/apt/apt.conf.d/99dm-frozen-snapshot
COPY apt/sources.list /etc/apt/sources.list

## COPY preserves the source file mode, and under a hardened umask (027/007) the
## checked-out files are 0660, which apt cannot read as any other user. Force the
## modes apt expects.
RUN chmod 0644 /etc/apt/sources.list.d/debian.sources \
      /etc/apt/apt.conf.d/99dm-frozen-snapshot \
      /etc/apt/sources.list

COPY derivative-maker-docker-setup /usr/bin
COPY build-data/docker-entrypoint.target /etc/systemd/system
COPY build-data/docker-entrypoint.service /etc/systemd/system
COPY build-data/docker-entrypoint-stop.sh /usr/bin

RUN /usr/bin/derivative-maker-docker-setup

FROM baseimage

LABEL org.opencontainers.image.authors="adrelanos@whonix.org"
LABEL org.opencontainers.image.description="Containerization of Kicksecure/derivative-maker"
LABEL org.opencontainers.image.title="derivative-maker-docker"
LABEL org.opencontainers.image.source="https://github.com/Kicksecure/derivative-maker"
LABEL org.opencontainers.image.documentation="https://www.kicksecure.com/wiki/Dev/Build_Documentation/VM"
LABEL org.opencontainers.image.version="1.0"
LABEL org.opencontainers.image.vendor="ENCRYPTED SUPPORT LLC"

COPY entrypoint.sh /usr/bin
COPY derivative-maker-docker-start /usr/bin

## COPY preserves the source file mode; under a hardened umask (e.g. 027/007)
## the checked-out scripts are 0770 (no world-exec), so once COPY'd 0770
## root:root they cannot be executed by the unprivileged 'user' the entrypoint
## drops to ('sudo -u user'), failing with "Permission denied". Force 0755 so
## the image is executable regardless of the builder's umask.
RUN chmod 0755 /usr/bin/entrypoint.sh /usr/bin/derivative-maker-docker-start

ENTRYPOINT ["/usr/bin/entrypoint.sh"]

CMD ["/bin/bash"]
