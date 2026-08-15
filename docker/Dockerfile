############################################
# Healthcheck binary
############################################
FROM golang:1.21.4-bookworm AS build_healthcheck
WORKDIR /app
COPY ./extra/healthcheck.go ./extra/healthcheck.go
RUN go build -o ./extra/healthcheck ./extra/healthcheck.go

############################################
# Self-contained Dockge-PVE base image
############################################
FROM node:22-bookworm-slim AS base

ARG DEBIAN_FRONTEND=noninteractive
ENV TERM=xterm-256color
ENV DOCKGE_TERMINAL_TYPE=xterm-256color

RUN apt-get update && apt-get install -y --no-install-recommends \
        bash \
        bash-completion \
        ca-certificates \
        curl \
        dnsutils \
        dumb-init \
        gnupg \
        iproute2 \
        iputils-ping \
        jq \
        less \
        lsof \
        mc \
        nano \
        ncurses-bin \
        ncurses-term \
        netcat-openbsd \
        openssh-client \
        procps \
        traceroute \
        unzip \
        vim \
        wget \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
        > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        docker-ce-cli \
        docker-compose-plugin \
    && npm install -g tsx \
    && printf '%s\n' \
        '' \
        '# Dockge-PVE interactive console defaults' \
        'if [ -z "$TERM" ] || [ "$TERM" = "console" ]; then export TERM=xterm-256color; fi' \
        'if [ -f /usr/share/bash-completion/bash_completion ]; then' \
        '    . /usr/share/bash-completion/bash_completion' \
        'fi' \
        >> /root/.bashrc \
    && rm -rf /var/lib/apt/lists/*

############################################
# Frontend build
############################################

FROM base AS frontend_build

WORKDIR /app

COPY ./package.json ./package.json
COPY ./package-lock.json ./package-lock.json

RUN npm ci

COPY . .

RUN npm run check-ts && npm run build:frontend

############################################
# Production Node dependencies
############################################
FROM base AS prod_deps
WORKDIR /app
COPY ./package.json ./package.json
COPY ./package-lock.json ./package-lock.json
RUN npm ci --omit=dev

############################################
# Main image
############################################
FROM base AS release
WORKDIR /app
COPY --from=build_healthcheck /app/extra/healthcheck /app/extra/healthcheck
COPY --from=prod_deps /app/node_modules /app/node_modules
COPY . .
COPY --from=frontend_build /app/frontend-dist /app/frontend-dist
RUN mkdir -p ./data

# Required by node-pty on current Node.js releases.
ENV UV_USE_IO_URING=0

VOLUME /app/data
EXPOSE 5001
HEALTHCHECK --interval=60s --timeout=30s --start-period=60s --retries=5 CMD extra/healthcheck
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["tsx", "./backend/index.ts"]
