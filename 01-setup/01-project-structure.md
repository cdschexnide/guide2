# Step 1: Create Project Structure

This guide walks through creating the initial project directory structure.

## Prerequisites

- Go 1.21+ installed
- Terminal/command line access
- Target directory: `/Users/home/Documents/databricks-function/target-structure/ingestion-function`

## Steps

### 1. Navigate to Target Directory

```bash
cd /Users/home/Documents/databricks-function/target-structure/ingestion-function
```

### 2. Create Directory Structure

Run these commands to create all necessary directories:

```bash
# Core directories
mkdir -p proto/google/api
mkdir -p proto/google/protobuf  
mkdir -p proto/protoc-gen-openapiv2/options
mkdir -p generated/proto
mkdir -p server/blade_server
mkdir -p server/job
mkdir -p server/utils
mkdir -p database/models
mkdir -p database/datasource
mkdir -p swagger
mkdir -p test/integration
mkdir -p test/testdata
```

### 3. Expected Structure

After running the commands, you should have:

```
ingestion-function/
├── proto/
│   ├── google/
│   │   ├── api/
│   │   └── protobuf/
│   └── protoc-gen-openapiv2/
│       └── options/
├── generated/
│   └── proto/
├── server/
│   ├── blade_server/
│   ├── job/
│   └── utils/
├── database/
│   ├── models/
│   └── datasource/
├── swagger/
└── test/
    ├── integration/
    └── testdata/
```

### 4. Initialize Go Module

```bash
# Initialize the Go module
go mod init blade-ingestion-service

# Create go.sum file
touch go.sum
```

### 5. Create Essential Files

Create these empty files that we'll populate later:

```bash
# Configuration files
touch .env.example
touch .gitignore
touch Makefile
touch README.md
touch Dockerfile
touch docker-compose.yml

# Proto file
touch proto/blade_ingestion.proto

# Server files
touch server/main.go
touch server/blade_server/server.go
touch server/blade_server/databricks.go
touch server/blade_server/blade_query_job.go

# Database files
touch database/database.go
touch database/models/blade_items.go
touch database/datasource/data_source.go

# Utils
touch server/utils/config.go
```

### 6. Verify Structure

Run this command to see your project structure:

```bash
find . -type f -name "*.go" -o -name "*.proto" -o -name ".*" -o -name "Makefile" -o -name "*.yml" -o -name "*.md" | sort
```

Expected output:
```
./.env.example
./.gitignore
./Dockerfile
./Makefile
./README.md
./database/database.go
./database/datasource/data_source.go
./database/models/blade_items.go
./docker-compose.yml
./go.mod
./go.sum
./proto/blade_ingestion.proto
./server/blade_server/blade_query_job.go
./server/blade_server/databricks.go
./server/blade_server/server.go
./server/main.go
./server/utils/config.go
```

## What Each Directory Contains

### `/proto`
- Protocol buffer definitions
- Google proto dependencies
- OpenAPI annotations

### `/generated`
- Auto-generated code from protoc
- DO NOT edit files here manually

### `/server`
- Main application code
- `blade_server/` - Core server implementation
- `job/` - Job system (will copy from cataloger)
- `utils/` - Utility functions and config

### `/database`
- Database connection logic
- GORM models
- Data source definitions

### `/swagger`
- Swagger UI static files
- Auto-generated API documentation

### `/test`
- Integration tests
- Test data and fixtures

## Troubleshooting

### Permission Denied
If you get permission errors:
```bash
sudo chown -R $(whoami) .
chmod -R 755 .
```

### Directory Already Exists
If directories already exist, that's fine. The `-p` flag prevents errors.

### Missing Directories
Double-check you're in the right location:
```bash
pwd
# Should output: /Users/home/Documents/databricks-function/target-structure/ingestion-function
```

## Next Steps

✅ Project structure created  
➡️ Continue to [02-dependencies.md](02-dependencies.md) to set up project dependencies