# Build stage
FROM golang:1.27.3-alpine AS build
RUN apk add --no-cache ca-certificates openssl
COPY updater/distrust-compromised-cas.sh /usr/local/bin/distrust-compromised-cas
RUN sh /usr/local/bin/distrust-compromised-cas && update-ca-certificates --fresh
WORKDIR /src

# An explicit ref is used by the updater. For ordinary `docker compose up
# --build`, resolve HEAD from the small .git metadata included in the build
# context so the API never falls back to a misleading "dev" version.
ARG GIT_COMMIT=

# Cache dependencies.
COPY backend/go.mod backend/go.sum ./
RUN go mod download

# Build a static binary (migrations are embedded via go:embed).
COPY backend/ ./
RUN set -eu; \
    resolved_ref="${GIT_COMMIT:-unknown}"; \
    short_ref="$(printf '%s' "${resolved_ref}" | cut -c1-7)"; \
    CGO_ENABLED=0 GOOS=linux go build -trimpath \
      -ldflags="-s -w -X github.com/wheresfrank/openfamily/backend/internal/config.buildVersion=${short_ref}" \
      -o /out/server ./cmd/server

# Runtime stage
FROM alpine:3.22
RUN apk add --no-cache ca-certificates openssl tzdata
COPY updater/distrust-compromised-cas.sh /usr/local/bin/distrust-compromised-cas
RUN sh /usr/local/bin/distrust-compromised-cas && update-ca-certificates --fresh
COPY --from=build /out/server /usr/local/bin/server
EXPOSE 8080
ENTRYPOINT ["server"]

