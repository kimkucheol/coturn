#
# Dockerfile of coturn/coturn:alpine Docker image.
#

ARG alpine_ver=3.14.3




#
# Stage 'dist-libprom' creates prometheus-client-c distribution.
#

# We compile prometheus-client-c from sources, because Alpine doesn't provide
# it as its package yet.
#
# TODO: Re-check this to be present in packages on next Alpine major version update.

# https://hub.docker.com/_/alpine
FROM alpine:${alpine_ver} AS dist-libprom

# Install tools for building.
RUN apk update \
 && apk add --no-cache \
        ca-certificates cmake g++ git make \
 && update-ca-certificates

# Install prometheus-client-c build dependencies.
RUN apk add --no-cache \
        libmicrohttpd-dev

# Prepare prometheus-client-c sources for building.
ARG prom_ver=0.1.3
RUN mkdir -p /build/ && cd /build/ \
 && git init \
 && git remote add origin https://github.com/digitalocean/prometheus-client-c \
 && git fetch --depth=1 origin "v${prom_ver}" \
 && git checkout FETCH_HEAD

# Build libprom.so from sources.
RUN mkdir -p /build/prom/build/ && cd /build/prom/build/ \
 && TEST=0 cmake -G "Unix Makefiles" \
                 -DCMAKE_INSTALL_PREFIX=/usr \
                 -DCMAKE_SKIP_BUILD_RPATH=TRUE \
                 -DCMAKE_C_FLAGS="-DPROM_LOG_ENABLE -g -O3" \
                 .. \
 && make

# Build libpromhttp.so from sources.
RUN mkdir -p /build/promhttp/build/ && cd /build/promhttp/build/ \
 # Fix compiler warning: -Werror=incompatible-pointer-types
 && sed -i 's/\&promhttp_handler/(MHD_AccessHandlerCallback)\&promhttp_handler/' \
           /build/promhttp/src/promhttp.c \
 && TEST=0 cmake -G "Unix Makefiles" \
                 -DCMAKE_INSTALL_PREFIX=/usr \
                 -DCMAKE_SKIP_BUILD_RPATH=TRUE \
                 -DCMAKE_C_FLAGS="-g -O3" \
                 .. \
 && make VERBOSE=1

# Install prometheus-client-c.
RUN LIBS_DIR=/out/$(dirname $(find /usr/ -name libc.so)) \
 && mkdir -p $LIBS_DIR/ \
 && cp -rf /build/prom/build/libprom.so \
           /build/promhttp/build/libpromhttp.so \
       $LIBS_DIR/ \
 && mkdir -p /out/usr/include/ \
 && cp -rf /build/prom/include/* \
           /build/promhttp/include/* \
       /out/usr/include/ \
 # Preserve license file.
 && mkdir -p /out/usr/share/licenses/prometheus-client-c/ \
 && cp /build/LICENSE /out/usr/share/licenses/prometheus-client-c/




#
# Stage 'dist-coturn' creates Coturn distribution.
#

# https://hub.docker.com/_/alpine
FROM alpine:${alpine_ver} AS dist-coturn

# Install tools for building.
RUN apk update \
 && apk add --no-cache \
        autoconf ca-certificates coreutils g++ git libtool make \
 && update-ca-certificates

# Install Coturn build dependencies.
RUN apk add --no-cache \
        linux-headers \
        libevent-dev \
        openssl-dev \
        postgresql-dev mariadb-connector-c-dev sqlite-dev \
        hiredis-dev \
        mongo-c-driver-dev \
        libmicrohttpd-dev

# Install prometheus-client-c distribution.
COPY --from=dist-libprom /out/ /

# Prepare local Coturn sources for building.
COPY CMakeLists.txt \
     configure \
     INSTALL \
     LICENSE LICENSE.OpenSSL \
     make-man.sh Makefile.in \
     postinstall.txt \
     README.turn* \
     /app/
COPY cmake/ /app/cmake/
COPY examples/ /app/examples/
COPY man/ /app/man/
COPY src/ /app/src/
COPY turndb/ /app/turndb/
WORKDIR /app/

# Use Coturn sources from Git if `coturn_git_ref` is specified.
ARG coturn_git_ref=HEAD
RUN if [ "${coturn_git_ref}" != 'HEAD' ]; then true \
 && rm -rf /app/* \
 && git init \
 && git remote add origin https://github.com/coturn/coturn \
 && git fetch --depth=1 origin "${coturn_git_ref}" \
 && git checkout FETCH_HEAD \
 && true; fi

# Build Coturn from sources.
# TODO: Remove this symlink with next Coturn release detecting MySQL libs correctly.
RUN ln -s /usr/lib/pkgconfig/libmariadb.pc /usr/lib/pkgconfig/mariadb.pc \
 && ./configure --prefix=/usr \
                --turndbdir=/var/lib/coturn \
                --disable-rpath \
                --sysconfdir=/etc/coturn \
                # No documentation included to keep image size smaller.
                --mandir=/tmp/coturn/man \
                --docsdir=/tmp/coturn/docs \
                --examplesdir=/tmp/coturn/examples \
 && make

# Install and configure Coturn.
RUN mkdir -p /out/ \
 && DESTDIR=/out make install \
 # Remove redundant files.
 && rm -rf /out/tmp/ \
 # Preserve license file.
 && mkdir -p /out/usr/share/licenses/coturn/ \
 && cp LICENSE /out/usr/share/licenses/coturn/ \
 # Remove default config file.
 && rm -f /out/etc/coturn/turnserver.conf.default

# Install helper tools of Docker image.
COPY docker/coturn/rootfs/ /out/
COPY docker/coturn/alpine/rootfs/ /out/
RUN chmod +x /out/usr/local/bin/docker-entrypoint.sh \
             /out/usr/local/bin/detect-external-ip.sh
RUN ln -s /usr/local/bin/detect-external-ip.sh \
          /out/usr/local/bin/detect-external-ip
RUN chown -R nobody:nogroup /out/var/lib/coturn/

# Re-export prometheus-client-c distribution.
COPY --from=dist-libprom /out/ /out/




#
# Stage 'runtime' creates final Docker image to use in runtime.
#

# https://hub.docker.com/_/alpine
FROM alpine:${alpine_ver} AS runtime

LABEL org.opencontainers.image.source="https://github.com/coturn/coturn"

# Update system packages.
RUN apk update \
 && apk upgrade \
 && apk add --no-cache ca-certificates \
 && update-ca-certificates \
 # Install Coturn dependencies.
 && apk add --no-cache \
        libevent \
        libcrypto1.1 libssl1.1 \
        libpq mariadb-connector-c sqlite-libs \
        hiredis \
        mongo-c-driver \
        libmicrohttpd \
 # Install `dig` tool for `detect-external-ip.sh`.
 && apk add --no-cache \
        bind-tools \
 # Cleanup unnecessary stuff.
 && rm -rf /var/cache/apk/*

# Install Coturn distribution.
COPY --from=dist-coturn /out/ /

# Allow non-root using privileged ports.
RUN apk add --no-cache libcap \
 && setcap CAP_NET_BIND_SERVICE=+ep /usr/bin/turnserver \
 # Cleanup unnecessary stuff.
 && apk del libcap \
 && rm -rf /var/cache/apk/*

USER nobody:nogroup

EXPOSE 3478 3478/udp

VOLUME ["/var/lib/coturn"]

ENTRYPOINT ["docker-entrypoint.sh"]

CMD ["--log-file=stdout", "--external-ip=$(detect-external-ip)"]
