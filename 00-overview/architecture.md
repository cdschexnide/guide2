# System Architecture

## Overview

The BLADE Ingestion Service is a microservice that ingests BLADE data from a mock Databricks server and catalogs it using the same patterns as infinityai-cataloger.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
│                    (gRPC Client / REST Client)                  │
└─────────────────┬───────────────────────┬──────────────────────┘
                  │                       │
                  ▼                       ▼
         ┌────────────────┐      ┌─────────────────┐
         │   gRPC Server  │      │  REST Gateway   │
         │   (Port 9090)  │      │  (Port 9091)    │
         └────────┬───────┘      └────────┬────────┘
                  │                       │
                  └───────────┬───────────┘
                              │
                  ┌───────────▼────────────┐
                  │   BLADE Ingestion      │
                  │      Service Core      │
                  │  ┌─────────────────┐  │
                  │  │ Request Handlers│  │
                  │  └────────┬────────┘  │
                  │           │           │
                  │  ┌────────▼────────┐  │
                  │  │   Job System    │  │
                  │  │ (Async Tasks)   │  │
                  │  └────────┬────────┘  │
                  └───────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │ PostgreSQL │    │   Mock      │    │   Catalog   │
    │  Database  │    │ Databricks  │    │   Service   │
    │            │    │   Server    │    │             │
    └────────────┘    └─────────────┘    └─────────────┘
```

## Core Components

### 1. API Layer (Dual API)

- **gRPC Server** (Port 9090)
  - Native gRPC interface
  - High performance for service-to-service communication
  - Protocol buffer based

- **REST Gateway** (Port 9091)
  - HTTP/JSON interface
  - Swagger UI documentation
  - Automatically generated from proto definitions

### 2. Service Core

- **Request Handlers**
  - Configuration management (data sources)
  - Query execution
  - Ingestion orchestration
  
- **Job System**
  - Async job execution
  - Pause/resume capability
  - Progress tracking
  - Job history

### 3. External Integrations

- **PostgreSQL Database**
  - Data source configuration
  - Job status tracking
  - Sync history
  - Error logging

- **Mock Databricks Server**
  - Simulates Databricks SQL endpoints
  - Returns mock BLADE data
  - Supports all BLADE data types

- **Catalog Service**
  - Final destination for ingested data
  - Multipart upload support
  - Metadata enrichment

## Data Flow

```
1. Client Request
   └─> API Layer (gRPC/REST)
       └─> Request Handler
           ├─> Sync Request: Create Job
           │   └─> Job Queue
           │       └─> Job Worker
           │           └─> Query Databricks
           │               └─> Process Results
           │                   └─> Upload to Catalog
           │                       └─> Update Status
           │
           └─> Direct Request: Execute Immediately
               └─> Query Databricks
                   └─> Return Results
```

## Key Design Patterns

### 1. Repository Pattern
- Database access through repository interfaces
- Clean separation of concerns
- Easy to mock for testing

### 2. Job Queue Pattern
- Async processing for long-running operations
- Worker pool for concurrent execution
- Graceful shutdown support

### 3. Dual API Pattern
- Single service definition
- Multiple access methods
- Consistent behavior

### 4. Configuration as Code
- Proto-first API design
- Generated code for consistency
- Type safety across languages

## Service Responsibilities

### What This Service DOES:
1. Manages BLADE data source configurations
2. Queries mock Databricks for BLADE data
3. Transforms data to catalog format
4. Uploads data to catalog service
5. Tracks ingestion progress and history
6. Provides async job management

### What This Service DOES NOT:
1. Store BLADE data (only configuration)
2. Process real Databricks clusters
3. Handle authentication (simplified for POC)
4. Implement catalog search functionality

## Technology Stack

- **Language**: Go 1.21+
- **API**: gRPC + REST (via grpc-gateway)
- **Database**: PostgreSQL 14+
- **ORM**: GORM
- **Job Queue**: Custom implementation (from cataloger)
- **Serialization**: Protocol Buffers
- **Documentation**: OpenAPI/Swagger

## Deployment Architecture

### Development
```
docker-compose up
├── blade-ingestion-service
├── postgres
├── mock-databricks
└── mock-catalog (optional)
```

### Production Considerations
- Kubernetes deployment ready
- Health check endpoints
- Metrics endpoints (future)
- Horizontal scaling via job workers

## Security Considerations

### Current (POC)
- Mock authentication tokens
- No TLS (local development)
- Basic error handling

### Production Ready
- OAuth2/JWT authentication
- TLS for all connections
- Secret management
- Audit logging

## Next Steps

Continue to [patterns.md](patterns.md) to understand the key patterns borrowed from infinityai-cataloger →