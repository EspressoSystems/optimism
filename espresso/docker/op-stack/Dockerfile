# Builds the Espresso enclave images for the OP stack batcher:
#
#   op-batcher-enclave-app    — inner image that Enclaver wraps into the EIF.
#                               Contains only the batcher binary and the
#                               enclave entrypoint (socat proxying, arg
#                               routing via nc).
#   op-batcher-enclave-target — runtime image for the TEE batcher
#                               (published as op-batcher-tee). Ships
#                               enclave-tools + enclaver; builds the EIF from
#                               a pre-built op-batcher-enclave-app image and
#                               runs it.
#
# Ported (trimmed) from optimism-espresso-integration
# espresso/docker/op-stack/Dockerfile: only the stages needed for the two
# enclave images are kept, and the builder stage drops the contract/deploy
# tooling (foundry, just, yq, dasel) that these targets never use.

ARG TARGET_BASE_IMAGE=alpine:3.22

FROM golang:1.26-alpine AS builder
# Add ARG to fix arm64 build.
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT

RUN apk add --no-cache \
  curl tar gzip make gcc g++ musl-dev \
  linux-headers git bash jq

# Copy and download Go dependencies
COPY ./go.mod /app/go.mod
COPY ./go.sum /app/go.sum
WORKDIR /app
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build go mod download

# Copy source code
COPY . /app

# Build arguments for git metadata
ARG GIT_COMMIT
ARG GIT_DATE

# Build op-batcher
FROM builder AS op-batcher-builder
ARG TARGETOS
ARG TARGETARCH
ARG GIT_COMMIT
ARG GIT_DATE
ARG OP_BATCHER_VERSION=v0.0.0
ENV GOOS=$TARGETOS GOARCH=$TARGETARCH GITCOMMIT=$GIT_COMMIT GITDATE=$GIT_DATE VERSION="$OP_BATCHER_VERSION"
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
  cd /app/op-batcher && mkdir -p bin && go build -v -ldflags "-X main.GitCommit=$GITCOMMIT -X main.GitDate=$GITDATE -X main.Version=$VERSION" -o ./bin/op-batcher ./cmd

# Build enclave-tools
FROM builder AS enclave-tools-builder
ARG TARGETOS
ARG TARGETARCH
ARG GIT_COMMIT
ARG GIT_DATE
ARG ENCLAVE_TOOLS_VERSION=v0.0.0
ENV GOOS=$TARGETOS GOARCH=$TARGETARCH GITCOMMIT=$GIT_COMMIT GITDATE=$GIT_DATE VERSION="$ENCLAVE_TOOLS_VERSION"
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
  cd /app/op-batcher && mkdir -p bin && go build -v -o ./bin/enclave-tools ./enclave-tools/cmd

# Download Aztec SRS once; COPY --from into each target that needs it.
FROM $TARGET_BASE_IMAGE AS aztec-srs
ADD "https://github.com/EspressoSystems/ark-srs/releases/download/v0.2.0/kzg10-aztec20-srs-1048584.bin" /aztec/kzg10-aztec20-srs-1048584.bin

# Inner image that Enclaver wraps into the EIF. Contains only the batcher
# binary and the enclave entrypoint (socat proxying, arg routing via nc).
FROM $TARGET_BASE_IMAGE AS op-batcher-enclave-app
RUN apk add libgcc bash socat trurl
ENV AZTEC_SRS_PATH=/aztec/kzg10-aztec20-srs-1048584.bin
COPY --from=aztec-srs /aztec/kzg10-aztec20-srs-1048584.bin /aztec/kzg10-aztec20-srs-1048584.bin
COPY --from=op-batcher-builder /app/op-batcher/bin/op-batcher /usr/local/bin/
COPY op-batcher/enclave-entrypoint.bash /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]

# Runtime image for TEE batcher.  Uses `enclave-tools build-eif` with the
# pre-built op-batcher-enclave-app image (eliminating a full rebuild from
# source at startup) then runs the resulting EIF via `enclave-tools run`.
FROM $TARGET_BASE_IMAGE AS op-batcher-enclave-target
RUN apk add libgcc docker bash jq curl wget
# Install enclaver for EIF creation
RUN curl -L https://github.com/enclaver-io/enclaver/releases/download/v0.6.1/enclaver-linux-x86_64-v0.6.1.tar.gz | tar xz --strip-components=1 -C /usr/local/bin enclaver-linux-x86_64-v0.6.1/enclaver
WORKDIR /app
# Deployment state (state.json with BatchAuthenticatorAddress) is mounted via
# docker-compose volumes at /app/deployment rather than baked into the image.
# run-enclave.sh — build EIF from pre-built app image + register PCR0 + run
COPY espresso/docker/op-batcher-tee/run-enclave.sh /app/run-enclave.sh
RUN chmod +x /app/run-enclave.sh
COPY --from=enclave-tools-builder /app/op-batcher/bin/enclave-tools /usr/local/bin/
CMD ["enclave-tools"]
