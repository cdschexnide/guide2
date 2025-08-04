# Dockerfile Implementation for BLADE Service

This guide explains how to create a production-ready multi-stage Dockerfile for the BLADE service after implementing the server components.

## Overview

The Dockerfile creates a containerized version of the BLADE service using a multi-stage build approach that separates the build environment from the runtime environment, resulting in a smaller, more secure final image.

## Prerequisites

- All server components implemented (main.go, server.go, database setup)
- Protobuf definitions in `proto/` directory
- Makefile with `proto` target for generating protobuf files
- Go modules configured with go.mod and go.sum

## Multi-Stage Dockerfile Implementation

### File: `Dockerfile`

```dockerfile
# Build stage - contains all build tools and dependencies
FROM golang:1.24-alpine AS builder

# Set build arguments for flexibility
ARG GO_VERSION=1.24
ARG BUILD_DATE
ARG VERSION=dev
ARG COMMIT_SHA

# Add metadata labels
LABEL maintainer="BLADE Service Team"
LABEL version="${VERSION}"
LABEL build-date="${BUILD_DATE}"
LABEL commit-sha="${COMMIT_SHA}"

# Set working directory
WORKDIR /app

# Install build dependencies
# protoc: Protocol buffer compiler
# protobuf-dev: Protocol buffer development files
# git: Required for Go module downloads
# make: Build automation tool
# ca-certificates: SSL certificates for HTTPS requests
RUN apk add --no-cache \
    protobuf \
    protobuf-dev \
    git \
    make \
    ca-certificates \
    tzdata

# Install Go protobuf generation tools
# These tools generate Go code from .proto files
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@latest && \
    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest && \
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest && \
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest

# Copy dependency files first for better layer caching
COPY go.mod go.sum ./

# Copy proto files (needed for code generation)
COPY proto/ ./proto/

# Create directories for generated files
RUN mkdir -p generated/proto swagger

# Download Go module dependencies
# This layer will be cached unless go.mod or go.sum changes
RUN go mod download

# Verify dependencies (security check)
RUN go mod verify

# Copy rest of source code
COPY . .

# Generate protobuf and gRPC code
RUN make proto

# Build the application
# CGO_ENABLED=0: Disable CGO for static binary
# GOOS=linux: Target Linux operating system
# -a: Force rebuilding of packages
# -installsuffix cgo: Add suffix to package installation directory
# -ldflags: Pass flags to linker for optimization
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -X main.version=${VERSION} -X main.buildDate=${BUILD_DATE} -X main.commitSHA=${COMMIT_SHA}" \
    -o minimal-blade-server \
    ./server/main.go

# Verify the binary was created and is executable
RUN ls -la minimal-blade-server && file minimal-blade-server

# Runtime stage - minimal image for production
FROM alpine:latest

# Install runtime dependencies
# ca-certificates: SSL certificates for HTTPS requests
# tzdata: Timezone data
RUN apk --no-cache add \
    ca-certificates \
    tzdata \
    dumb-init

# Create non-root user for security
RUN adduser -D -s /bin/sh -u 1001 blade

# Set working directory
WORKDIR /app

# Copy binary from builder stage
COPY --from=builder /app/minimal-blade-server .

# Copy swagger files for API documentation
COPY --from=builder /app/swagger ./swagger

# Create necessary directories with proper permissions
RUN mkdir -p logs data && \
    chown -R blade:blade /app

# Switch to non-root user
USER blade

# Expose ports
# 9090: gRPC server port
# 9091: HTTP REST gateway port
EXPOSE 9090 9091

# Health check to ensure service is running
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:9091/healthz || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start the application
CMD ["./minimal-blade-server"]
```

## Build Arguments and Environment

### Build-time Arguments

```dockerfile
ARG GO_VERSION=1.24
ARG BUILD_DATE
ARG VERSION=dev
ARG COMMIT_SHA
```

**Usage in build:**
```bash
docker build \
  --build-arg VERSION=1.0.0 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg COMMIT_SHA=$(git rev-parse HEAD) \
  -t minimal-blade-service:1.0.0 \
  .
```

### Runtime Environment Variables

The container expects these environment variables:

```bash
# Database configuration
PGHOST=postgres
PGPORT=5432
PG_DATABASE=blade_db
PGUSER=blade_user
PGPASSWORD=blade_password
DB_SCHEMA=public

# Service configuration
GRPC_PORT=9090
REST_PORT=9091
LOG_LEVEL=info

# Feature flags
ENABLE_SWAGGER_UI=true
SEED_INITIAL_DATA=false

# Connection pooling
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_MAX_LIFETIME_MINUTES=60
```

## Dockerfile Optimization Strategies

### 1. Layer Caching Optimization

```dockerfile
# Copy dependency files BEFORE copying source code
COPY go.mod go.sum ./
RUN go mod download

# Copy source code AFTER dependencies
COPY . .
```

**Benefits:**
- Dependencies layer cached unless go.mod/go.sum changes
- Faster rebuilds when only source code changes

### 2. Multi-Stage Build Benefits

```dockerfile
FROM golang:1.24-alpine AS builder  # Build stage
# ... build process ...

FROM alpine:latest                  # Runtime stage
COPY --from=builder /app/binary .   # Copy only the binary
```

**Benefits:**
- Smaller final image (no build tools)
- More secure (fewer attack vectors)
- Faster deployment and startup

### 3. Static Binary Compilation

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo
```

**Benefits:**
- No dynamic library dependencies
- Works with minimal base images
- Smaller image size

## Security Best Practices

### 1. Non-Root User

```dockerfile
RUN adduser -D -s /bin/sh -u 1001 blade
USER blade
```

**Benefits:**
- Principle of least privilege
- Reduces container escape risks
- Meets security compliance requirements

### 2. Minimal Base Image

```dockerfile
FROM alpine:latest  # Minimal Linux distribution
```

**Benefits:**
- Smaller attack surface
- Fewer vulnerabilities
- Faster image pulls

### 3. Dependency Verification

```dockerfile
RUN go mod verify  # Verify module checksums
```

**Benefits:**
- Ensures dependency integrity
- Prevents supply chain attacks
- Validates module checksums

## Production-Ready Features

### 1. Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:9091/healthz || exit 1
```

**Benefits:**
- Container orchestration can detect unhealthy containers
- Automatic restart of failed containers
- Load balancer integration

### 2. Signal Handling

```dockerfile
ENTRYPOINT ["dumb-init", "--"]
```

**Benefits:**
- Proper signal forwarding to application
- Graceful shutdown handling
- Zombie process reaping

### 3. Build Metadata

```dockerfile
LABEL maintainer="BLADE Service Team"
LABEL version="${VERSION}"
LABEL build-date="${BUILD_DATE}"
```

**Benefits:**
- Image traceability
- Version tracking
- Deployment automation support

## Advanced Dockerfile Features

### 1. Development Dockerfile

Create `Dockerfile.dev` for development:

```dockerfile
FROM golang:1.24-alpine

WORKDIR /app

# Install development tools
RUN apk add --no-cache \
    protobuf \
    protobuf-dev \
    git \
    make \
    air \
    dlv

# Install protoc plugins
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@latest && \
    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest && \
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest && \
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest

# Install live reload tool
RUN go install github.com/cosmtrek/air@latest

# Install debugger
RUN go install github.com/go-delve/delve/cmd/dlv@latest

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Generate proto files
RUN make proto

EXPOSE 9090 9091 2345

# Use air for live reload in development
CMD ["air"]
```

### 2. Build Script Integration

Create `scripts/build.sh`:

```bash
#!/bin/bash
set -e

# Build variables
VERSION=${VERSION:-$(git describe --tags --always --dirty)}
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
COMMIT_SHA=$(git rev-parse HEAD)

echo "Building BLADE Service version: $VERSION"

# Build production image
docker build \
  --build-arg VERSION="$VERSION" \
  --build-arg BUILD_DATE="$BUILD_DATE" \
  --build-arg COMMIT_SHA="$COMMIT_SHA" \
  -t minimal-blade-service:"$VERSION" \
  -t minimal-blade-service:latest \
  .

echo "Build completed successfully"
```

### 3. .dockerignore File

Create `.dockerignore` to exclude unnecessary files:

```
# Git
.git
.gitignore
.gitattributes

# Documentation
*.md
docs/

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Build artifacts
*.log
*.tmp
tmp/

# Test files
*_test.go
test/
coverage.out

# Development files
.env.local
.env.development
air.toml

# Docker files
Dockerfile.dev
docker-compose.override.yml
```

## Image Size Optimization

### 1. Use Specific Alpine Version

```dockerfile
FROM alpine:3.18  # Specific version instead of 'latest'
```

### 2. Combine RUN Commands

```dockerfile
RUN apk add --no-cache ca-certificates tzdata && \
    adduser -D -s /bin/sh -u 1001 blade && \
    mkdir -p logs data
```

### 3. Clean Package Cache

```dockerfile
RUN apk add --no-cache package && \
    rm -rf /var/cache/apk/*
```

## Build Commands

### 1. Standard Build

```bash
docker build -t minimal-blade-service .
```

### 2. Build with Version

```bash
docker build \
  --build-arg VERSION=1.0.0 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg COMMIT_SHA=$(git rev-parse HEAD) \
  -t minimal-blade-service:1.0.0 \
  .
```

### 3. Development Build

```bash
docker build -f Dockerfile.dev -t minimal-blade-service:dev .
```

### 4. Multi-Platform Build

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t minimal-blade-service:1.0.0 \
  --push \
  .
```

## Testing the Docker Image

### 1. Basic Functionality Test

```bash
# Run the container
docker run -d \
  --name blade-test \
  -p 9090:9090 \
  -p 9091:9091 \
  -e PGHOST=host.docker.internal \
  -e PGPASSWORD=password \
  minimal-blade-service:latest

# Test health endpoint
curl http://localhost:9091/healthz

# Check logs
docker logs blade-test

# Clean up
docker stop blade-test
docker rm blade-test
```

### 2. Image Security Scan

```bash
# Scan for vulnerabilities (requires Docker Desktop or Trivy)
docker scan minimal-blade-service:latest

# Or with Trivy
trivy image minimal-blade-service:latest
```

### 3. Image Analysis

```bash
# Check image size
docker images minimal-blade-service

# Analyze layers
docker history minimal-blade-service:latest

# Inspect image details
docker inspect minimal-blade-service:latest
```

## Troubleshooting

### 1. Build Failures

**Proto generation fails:**
```bash
# Check if proto files exist
docker run --rm -v $(pwd):/app golang:1.24-alpine ls -la /app/proto

# Check protoc installation
docker run --rm golang:1.24-alpine protoc --version
```

**Binary build fails:**
```bash
# Check Go module issues
docker run --rm -v $(pwd):/app -w /app golang:1.24-alpine go mod verify
```

### 2. Runtime Issues

**Container exits immediately:**
```bash
# Check for missing dependencies
docker run --rm minimal-blade-service:latest ldd minimal-blade-server

# Check binary permissions
docker run --rm minimal-blade-service:latest ls -la minimal-blade-server
```

**Health check fails:**
```bash
# Test health endpoint manually
docker exec blade-container wget -qO- http://localhost:9091/healthz
```

## Deployment Considerations

### 1. Resource Limits

```yaml
# In docker-compose.yml or Kubernetes
resources:
  limits:
    memory: 512Mi
    cpu: 500m
  requests:
    memory: 256Mi
    cpu: 250m
```

### 2. Environment Configuration

```bash
# Use environment files for different environments
docker run --env-file .env.production minimal-blade-service:latest
```

### 3. Volume Mounts

```bash
# Mount logs directory for persistence
docker run -v ./logs:/app/logs minimal-blade-service:latest
```

This Dockerfile implementation provides a production-ready, secure, and optimized container for the BLADE service with proper build practices, security measures, and operational features.
