
# ---------- BUILDER STAGE ----------
# Use full Debian only for BUILD; all tooling stays here.
FROM node:22-bookworm-slim AS build

# Set Python for node-gyp (builder only)
ENV PYTHON=/usr/bin/python3
ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

# Install native build deps (builder only)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

# Use non-root for reproducible caches
USER node
WORKDIR /app

# Yarn core files
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./
COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./

# Unpack skeleton and install prod deps
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

# Cache yarn downloads per user
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && yarn cache clean

# Copy examples only if needed (optional, comment out to slim)
# COPY --chown=node:node examples ./examples

# Copy backend bundle (compiled JS)
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

# ---------- RUNTIME STAGE ----------
# Distroless Node 22 (nonroot). No shell, no apt, minimal attack surface.
FROM gcr.io/distroless/nodejs22:nonroot

# Distroless runs as nonroot already
WORKDIR /app

# Copy only the built app + configs from builder
COPY --from=build /app /app

# Environment for runtime
ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

# Expose if your platform needs it (optional purely informational)
# EXPOSE 7007

# Start Backstage backend
CMD ["/app/packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
