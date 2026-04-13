FROM debian:trixie-slim AS engine

RUN apt update && apt upgrade -y && apt -y --no-install-recommends install aptitude
RUN aptitude -y --without-recommends install git build-essential ca-certificates libsdl2-dev \
    libfreetype6-dev libopus-dev libbz2-dev libvorbis-dev libopusfile-dev libogg-dev

WORKDIR /xash

COPY ./xash3d-fwgs .

RUN ./waf configure -T release -8 -d --enable-openmp \
    && ./waf build

FROM golang:1.25.1 AS go

WORKDIR /go

# cria pasta pkg
RUN mkdir -p /go/pkg

# copia todas as libs estáticas do build do xash
COPY --from=engine /xash/build/ /tmp/xash-build/

# move TODAS as .a para /go/pkg
RUN find /tmp/xash-build -name "*.a" -exec cp {} /go/pkg/ \;

# copia código go
COPY go.mod .
COPY go.sum .
COPY go.work .
COPY examples ./examples
COPY pkg ./pkg

WORKDIR /go/examples/webrtc-hl

ENV CGO_CFLAGS="-fopenmp"
ENV CGO_LDFLAGS="-L/go/pkg -fopenmp"

RUN go mod download && go build -o webrtc-hl .


FROM debian:trixie-slim AS hlds

RUN apt update && apt upgrade -y && apt -y --no-install-recommends install aptitude
RUN aptitude -y --without-recommends install git ca-certificates cmake build-essential libsdl2-dev

WORKDIR /
RUN git clone https://github.com/FWGS/hlsdk-portable.git
WORKDIR /hlsdk-portable
RUN ./waf configure -T release -8 && \
    ./waf

ARG hlds_build=8308
ARG hlds_url="https://github.com/DevilBoy-eXe/hlds/releases/download/$hlds_build/hlds_build_$hlds_build.zip"

RUN groupadd -r xash && useradd -r -g xash -m -d /opt/xash xash
RUN usermod -a -G games xash

RUN apt-get -y update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    unzip \
    && apt-get -y clean

USER xash
WORKDIR /opt/xash
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN mkdir -p /opt/xash/xashds

RUN curl -sLJO "$hlds_url" \
    && unzip "hlds_build_$hlds_build.zip" -d "/opt/xash/hlds_build_$hlds_build" \
    && cp -R "hlds_build_$hlds_build/hlds"/* xashds/ \
    && rm -rf "hlds_build_$hlds_build" "hlds_build_$hlds_build.zip"

# Fix warnings:
# couldn't exec listip.cfg
# couldn't exec banned.cfg
RUN touch /opt/xash/xashds/valve/listip.cfg
RUN touch /opt/xash/xashds/valve/banned.cfg

# Remove cstrike game directory, because it's not needed
WORKDIR /opt/xash/xashds
RUN rm -rf ./cstrike

# Copy default config
COPY configs/valve valve

FROM debian:trixie-slim AS final

ENV XASH3D_BASEDIR=/xashds

RUN apt-get update && apt-get install -y --no-install-recommends \
    libgcc-s1 \
    libstdc++6 \
    libgomp1 \
    ca-certificates \
    openssl \
    && apt-get clean

RUN groupadd xashds && useradd -m -g xashds xashds
USER xashds
WORKDIR /xashds
ENV LD_LIBRARY_PATH=/xashds

COPY --from=hlds /opt/xash/xashds .
COPY --from=hlds /hlsdk-portable/build/dlls/hl_amd64.so dlls/hl_amd64.so
COPY --from=go /go/examples/webrtc-hl/webrtc-hl ./xash
COPY --from=engine /xash/build/filesystem/filesystem_stdio.so ./filesystem_stdio.so
EXPOSE 27015/udp
EXPOSE 27016/tcp

# Start server
ENTRYPOINT ["./xash", "-port", "27015", "+ip", "0.0.0.0"]

# Default start parameters
CMD ["+map crossfire"]