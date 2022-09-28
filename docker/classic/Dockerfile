FROM alpine AS builder-base
# General Build System:
RUN apk -U add \
        git \
        build-base \
        autoconf \
        automake \
        libtool \
        dbus \
        su-exec \
        alsa-lib-dev \
        libdaemon-dev \
        popt-dev \
        mbedtls-dev \
        soxr-dev \
        avahi-dev \
        libconfig-dev \
        libsndfile-dev \
        mosquitto-dev \
        xmltoman

# ALAC Build System:
FROM builder-base AS builder-alac

RUN git clone https://github.com/mikebrady/alac
WORKDIR alac
RUN autoreconf -fi
RUN ./configure
RUN make
RUN make install

# Shairport Sync Build System:
FROM builder-base AS builder-sps

# This will be modified by the Github Action Workflow and is required
# to ensure the correct branch is being used.
ARG SHAIRPORT_SYNC_BRANCH
RUN test -n "$SHAIRPORT_SYNC_BRANCH"

COPY --from=builder-alac /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=builder-alac /usr/local/lib/pkgconfig/alac.pc /usr/local/lib/pkgconfig/alac.pc
COPY --from=builder-alac /usr/local/include /usr/local/include

WORKDIR /shairport-sync
COPY . .
RUN git checkout "$SHAIRPORT_SYNC_BRANCH"
RUN autoreconf -fi
RUN ./configure \
        --with-alsa \
        --with-dummy \
        --with-pipe \
        --with-stdout \
        --with-avahi \
        --with-ssl=mbedtls \
        --with-soxr \
        --sysconfdir=/etc \
        --with-dbus-interface \
        --with-mpris-interface \
        --with-mqtt-client \
        --with-apple-alac \
        --with-convolution
RUN make -j $(nproc)
RUN make install

# Shairport Sync Runtime System:
FROM alpine

RUN apk -U add \
        alsa-lib \
        dbus \
        popt \
        glib \
        mbedtls \
        soxr \
        avahi \
        avahi-tools \
        libconfig \
        libsndfile \
        mosquitto-libs \
        su-exec \
        libgcc \
        libgc++

RUN rm -rf /lib/apk/db/*

COPY --from=builder-alac /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=builder-sps /etc/shairport-sync* /etc/
COPY --from=builder-sps /etc/dbus-1/system.d/shairport-sync-dbus.conf /etc/dbus-1/system.d/
COPY --from=builder-sps /etc/dbus-1/system.d/shairport-sync-mpris.conf /etc/dbus-1/system.d/
COPY --from=builder-sps /usr/local/bin/shairport-sync /usr/local/bin/shairport-sync

# Create non-root user for running the container -- running as the user 'shairport-sync' also allows
# Shairport Sync to provide the D-Bus and MPRIS interfaces within the container

RUN addgroup shairport-sync 
RUN adduser -D shairport-sync -G shairport-sync

# Add the shairport-sync user to the pre-existing audio group, which has ID 29, for access to the ALSA stuff
RUN addgroup -g 29 docker_audio && addgroup shairport-sync docker_audio

COPY ./docker/classic/start.sh /
RUN chmod +x /start.sh

ENTRYPOINT [ "/start.sh" ]