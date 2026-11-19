# Dockerfile for Debian-based event publisher sidecar

# Use official Debian slim image as base
FROM debian:bookworm-slim

# Install necessary runtime dependencies
RUN apt-get update \
    && apt-get install -y \
    curl \
    jq \
    hostname \
    awk \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -ms /bin/bash nonroot

# Set working directory
WORKDIR /app

# Build context is the package directory (packages/sendloki/)
COPY sendloki.sh /app/sendloki

# Set executable permission
RUN chmod +x /app/sendloki

# Set entrypoint
ENTRYPOINT ["/app/sendloki"]

# Set user
USER nonroot