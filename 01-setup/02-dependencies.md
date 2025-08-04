# Step 2: Install Dependencies

This guide covers setting up all project dependencies and creating initial configuration files.

## 1. Create go.mod with Dependencies

Replace the contents of `go.mod`:

```go
module blade-ingestion-service

go 1.21

require (
    // gRPC and Protocol Buffers
    google.golang.org/grpc v1.59.0
    google.golang.org/protobuf v1.31.0
    github.com/grpc-ecosystem/grpc-gateway/v2 v2.18.0
    
    // Database
    gorm.io/gorm v1.25.5
    gorm.io/driver/postgres v1.5.3
    
    // Configuration
    github.com/joho/godotenv v1.5.1
    
    // HTTP and middleware
    github.com/rs/cors v1.10.1
    github.com/gorilla/mux v1.8.0
    
    // Utilities
    github.com/google/uuid v1.4.0
    github.com/stretchr/testify v1.8.4
    
    // Job scheduling (for job system)
    github.com/go-co-op/gocron/v2 v2.1.2
)
```

## 2. Download Dependencies

```bash
# Download all dependencies
go mod download

# Tidy up dependencies
go mod tidy
```

## 3. Create .gitignore

Create `.gitignore` with common exclusions:

```gitignore
# Binaries
*.exe
*.dll
*.so
*.dylib
bin/
dist/

# Test binary, built with `go test -c`
*.test

# Output of the go coverage tool
*.out

# Dependency directories
vendor/

# Go workspace file
go.work

# Environment files
.env
.env.local
.env.*.local

# IDE specific files
.idea/
.vscode/
*.swp
*.swo
*~

# OS specific
.DS_Store
Thumbs.db

# Generated files
/generated/
*.pb.go
*.pb.gw.go

# Logs
*.log

# Temporary files
*.tmp
*.temp

# Build artifacts
/builds/

# Swagger UI maps (large files)
swagger/*.map
```

## 4. Create .env.example

Create `.env.example` with all configuration options:

```bash
# Server Configuration
HOST=0.0.0.0
GRPC_PORT=9090
REST_PORT=9091

# Database Configuration
PGHOST=localhost
PGPORT=5432
PG_USER=blade_user
APP_DB_ADMIN_PASSWORD=blade_password
PG_DATABASE=blade_ingestion
DB_SCHEMA_NAME=public

# Mock Databricks Configuration
MOCK_DATABRICKS_URL=http://localhost:8080
MOCK_DATABRICKS_TOKEN=mock-token
MOCK_DATABRICKS_WAREHOUSE_ID=test-warehouse
MOCK_REQUEST_TIMEOUT=30s

# Catalog Service Configuration
CATALOG_URL=http://localhost:8092
CATALOG_AUTH_TOKEN=test-catalog-token
CATALOG_TIMEOUT=60s
CATALOG_BATCH_SIZE=10
CATALOG_RETRY_ATTEMPTS=3

# Data Processing Configuration
DEFAULT_CLASSIFICATION=UNCLASSIFIED
MAX_RECORDS_PER_QUERY=1000
ENABLE_DATA_VALIDATION=true

# Performance Configuration
CONCURRENT_UPLOADS=5
RATE_LIMIT_PER_SECOND=10
PROCESSING_TIMEOUT=5m

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json

# Feature Flags
USE_SSL=false
ENABLE_SWAGGER_UI=true
ENABLE_METRICS=false
```

## 5. Create Initial Makefile

Create a basic `Makefile`:

```makefile
.PHONY: all proto build run test clean help

# Default target
all: proto build

# Help target
help:
	@echo "Available targets:"
	@echo "  make proto    - Generate protobuf files"
	@echo "  make build    - Build the server binary"
	@echo "  make run      - Run the server"
	@echo "  make test     - Run tests"
	@echo "  make clean    - Clean generated files"
	@echo "  make docker   - Build Docker image"
	@echo "  make deps     - Install dependencies"

# Install dependencies
deps:
	@echo "Installing dependencies..."
	go mod download
	go mod tidy

# Generate proto files (we'll implement this later)
proto:
	@echo "Proto generation will be implemented in step 3..."

# Build the server
build:
	@echo "Building server..."
	go build -o bin/blade-server ./server/main.go

# Run the server
run: build
	@echo "Starting server..."
	./bin/blade-server

# Run tests
test:
	@echo "Running tests..."
	go test -v ./...

# Clean generated files
clean:
	@echo "Cleaning..."
	rm -rf bin/ generated/*.pb.go generated/*.pb.gw.go

# Docker build
docker:
	@echo "Building Docker image..."
	docker build -t blade-ingestion-service .

# Development mode with hot reload
dev:
	@echo "Starting in development mode..."
	go run ./server/main.go
```

## 6. Create Initial README.md

Create a basic `README.md`:

```markdown
# BLADE Ingestion Service

A microservice for ingesting BLADE data from Databricks into a catalog system.

## Quick Start

1. Copy environment file:
   ```bash
   cp .env.example .env
   ```

2. Install dependencies:
   ```bash
   make deps
   ```

3. Generate proto files:
   ```bash
   make proto
   ```

4. Run the service:
   ```bash
   make run
   ```

## Development

See the implementation guide in `blade-ingestion-implementation-guide/` for detailed instructions.

## API Documentation

- gRPC: localhost:9090
- REST: localhost:9091
- Swagger UI: http://localhost:9091/swagger-ui/
```

## 7. Install Protocol Buffer Compiler

You'll need `protoc` to generate code from proto files:

### macOS:
```bash
brew install protobuf
```

### Linux:
```bash
# Download and install protoc
PROTOC_ZIP=protoc-21.12-linux-x86_64.zip
curl -OL https://github.com/protocolbuffers/protobuf/releases/download/v21.12/$PROTOC_ZIP
unzip -o $PROTOC_ZIP -d /usr/local bin/protoc
unzip -o $PROTOC_ZIP -d /usr/local 'include/*'
rm -f $PROTOC_ZIP
```

### Verify Installation:
```bash
protoc --version
# Should output: libprotoc 3.21.12 or similar
```

## 8. Install Go Proto Plugins

Install the Go plugins for protoc:

```bash
# Install protoc plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest

# Add Go bin to PATH if not already there
export PATH="$PATH:$(go env GOPATH)/bin"
```

## 9. Verify Dependencies

Run these commands to verify everything is installed:

```bash
# Check Go version
go version

# Check protoc
protoc --version

# Check Go plugins
which protoc-gen-go
which protoc-gen-go-grpc
which protoc-gen-grpc-gateway
which protoc-gen-openapiv2

# Check go.mod
cat go.mod | grep -E "grpc|gorm|gateway"
```

## Troubleshooting

### Missing protoc
If `protoc` is not found:
- Make sure it's in your PATH
- Try using the full path: `/usr/local/bin/protoc`

### Missing Go plugins
If protoc plugins are not found:
```bash
# Add to your shell profile (.bashrc, .zshrc, etc.)
export PATH=$PATH:$(go env GOPATH)/bin
```

### Module errors
If you get module errors:
```bash
go clean -modcache
go mod download
```

## Next Steps

✅ Dependencies installed  
✅ Configuration files created  
➡️ Continue to [03-configuration.md](03-configuration.md) to set up environment configuration