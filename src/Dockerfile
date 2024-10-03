FROM alpine:latest

LABEL author="EinsWlanKabel" maintainer="EinsWlanKabel"
LABEL org.opencontainers.image.source="https://github.com/EinsWlanKabel/server/tree/pterotest"
LABEL org.opencontainers.image.licenses=MIT

# Create a non-root user and set environment variables
RUN adduser -D -u 1000 container \
    && apk add --no-cache ca-certificates \
    && mkdir -p /home/container

USER container
ENV USER=container HOME=/home/container

# Set the working directory
WORKDIR /home/container

# Copy the entrypoint script
COPY ./entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Install ffmpeg from an external source
FROM ghcr.io/commandcracker/ffmpeg:latest AS ffmpeg

# Build the Sanjuuni tool
FROM ffmpeg AS sanjuuni

ENV SANJUUNI_VERSION=49cb275d4ef64d2bee3d5d2cbc5baf47787bedc2
ARG SANJUUNI_SHA512SUM="22c85f9a5c16c0acb4dde971b64f53e765a8d607065e7a25115103925124e33a0680da3790e85c4343b7d320a0170b6ae096ec7f2b48b22582cf96681fd3a78c *sanjuuni.tar.gz"

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

RUN set -eux; \
    apk add --no-cache g++ zlib-dev poco-dev make; \
    wget -q --output-document=sanjuuni.tar.gz https://github.com/MCJack123/sanjuuni/archive/${SANJUUNI_VERSION}.tar.gz; \
    echo "${SANJUUNI_SHA512SUM}" | sha512sum -c -; \
    mkdir --parents sanjuuni; \
    tar --extract --directory sanjuuni --strip-components=1 --file=sanjuuni.tar.gz; \
    rm sanjuuni.tar.gz;

WORKDIR /home/container/sanjuuni

RUN set -eux; \
    ./configure; \
    make

# Build the Python environment
FROM ghcr.io/commandcracker/alpine-pypy3.10-pip:3.20.1-pypy-7.3.14-pip-24.1.1 AS builder

WORKDIR /home/container/

COPY requirements.txt ./
COPY youcube ./youcube
COPY compile.py ./

RUN set -eux; \
    apk add --no-cache build-base; \
    pip install --no-cache-dir -U setuptools -r requirements.txt; \
    python3 compile.py; \
    pip uninstall pip -y

# Final runtime image
FROM alpine:3.20.1

# Set the working directory
WORKDIR /opt/server

RUN set -eux; \
    apk add --no-cache --update \
        openssl libffi libbz2 poco libgcc libstdc++ ca-certificates libgomp expat; \
    apk add --no-cache --repository=http://dl-cdn.alpinelinux.org/alpine/v3.18/community libssl1.1=1.1.1u-r1 libcrypto1.1=1.1.1u-r1; \
    chown -R 1000:1000 /opt/server/

# Copy necessary files from previous stages
COPY --from=builder /opt/pypy /opt/pypy
COPY --from=ffmpeg /usr/local /usr/local
COPY --from=sanjuuni /home/container/sanjuuni/sanjuuni /usr/local/bin/

# Environment variables
ENV HOST=0.0.0.0 \
    PATH="/opt/pypy/bin:$PATH" \
    LD_LIBRARY_PATH="/usr/local/lib:/usr/local/lib64" \
    XDG_CACHE_HOME="/opt/server/.yt-dlp-cache" \
    SANIC_NO_UVLOOP=true

# Switch to non-root user
USER 1000:1000

# Copy the compiled youcube files
COPY --from=builder /home/container/youcube/__pycache__ /opt/server

# Set the entrypoint for the container
#ENTRYPOINT ["python3", "youcube.pyc"]
CMD [ "/bin/ash", "/entrypoint.sh" ]
