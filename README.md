# BLADE Ingestion Service Implementation Guide

This guide provides step-by-step instructions to create a minimal replica of the infinityai-cataloger service containing only BLADE query functionality.

## 📁 Guide Structure

### Phase 1: Foundation
- **[00-overview/](00-overview/)** - Architecture overview and patterns
  - [architecture.md](00-overview/architecture.md) - System architecture
  - [patterns.md](00-overview/patterns.md) - Key patterns from cataloger
  - [data-flow.md](00-overview/data-flow.md) - Data flow through the system

### Phase 2: Project Setup
- **[01-setup/](01-setup/)** - Initial project setup
  - [01-project-structure.md](01-setup/01-project-structure.md) - Create directory structure
  - [02-dependencies.md](01-setup/02-dependencies.md) - Install dependencies
  - [03-configuration.md](01-setup/03-configuration.md) - Environment configuration
  - [04-file-migration.md](01-setup/04-file-migration.md) - Migrate existing files

### Phase 3: API Definition
- **[02-proto/](02-proto/)** - Protocol buffer implementation
  - [01-proto-setup.md](02-proto/01-proto-setup.md) - Proto dependencies setup
  - [02-blade-proto.md](02-proto/02-blade-proto.md) - Main proto definition
  - [03-generate-code.md](02-proto/03-generate-code.md) - Code generation

### Phase 4: Core Server
- **[03-server/](03-server/)** - Server implementation
  - [01-main-server.md](03-server/01-main-server.md) - Main server setup
  - [02-server-struct.md](03-server/02-server-struct.md) - Server struct and initialization
  - [03-grpc-gateway.md](03-server/03-grpc-gateway.md) - Dual API setup

### Phase 5: Job System
- **[04-job-system/](04-job-system/)** - Async job implementation
  - [01-copy-job-framework.md](04-job-system/01-copy-job-framework.md) - Copy from cataloger
  - [02-blade-query-job.md](04-job-system/02-blade-query-job.md) - BLADE query job
  - [03-job-endpoints.md](04-job-system/03-job-endpoints.md) - Job control endpoints

### Phase 6: Database
- **[05-database/](05-database/)** - Database setup
  - [01-database-connection.md](05-database/01-database-connection.md) - PostgreSQL setup
  - [02-models.md](05-database/02-models.md) - Data models
  - [03-migrations.md](05-database/03-migrations.md) - Database migrations

### Phase 7: Business Logic
- **[06-handlers/](06-handlers/)** - Request handlers
  - [01-configuration-handlers.md](06-handlers/01-configuration-handlers.md) - Data source config
  - [02-query-handlers.md](06-handlers/02-query-handlers.md) - Query endpoints
  - [03-ingestion-handlers.md](06-handlers/03-ingestion-handlers.md) - Ingestion logic
  - [04-catalog-upload.md](06-handlers/04-catalog-upload.md) - Catalog integration

### Phase 8: Testing
- **[07-testing/](07-testing/)** - Testing strategy
  - [01-unit-tests.md](07-testing/01-unit-tests.md) - Unit test setup
  - [02-integration-tests.md](07-testing/02-integration-tests.md) - Integration tests
  - [03-mock-services.md](07-testing/03-mock-services.md) - Mock Databricks

### Phase 9: Deployment
- **[08-deployment/](08-deployment/)** - Deployment setup
  - [01-dockerfile.md](08-deployment/01-dockerfile.md) - Docker configuration
  - [02-docker-compose.md](08-deployment/02-docker-compose.md) - Multi-service setup
  - [03-makefile.md](08-deployment/03-makefile.md) - Build automation

### Reference
- **[reference/](reference/)** - Reference materials
  - [code-snippets.md](reference/code-snippets.md) - Reusable code
  - [troubleshooting.md](reference/troubleshooting.md) - Common issues
  - [api-examples.md](reference/api-examples.md) - API usage examples

## 🚀 Quick Start

1. Start with **[00-overview/architecture.md](00-overview/architecture.md)** to understand the system
2. Follow **[01-setup/](01-setup/)** guides in order to create the project
3. Continue through each phase sequentially
4. Use **[reference/](reference/)** for help with specific issues

## 📋 Prerequisites

- Go 1.21+
- PostgreSQL 14+
- Docker & Docker Compose
- protoc (Protocol Buffer compiler)
- grpcurl (for testing)

## 🎯 End Goal

By following this guide, you'll create:
- A fully functional BLADE ingestion service
- Matching infinityai-cataloger architecture patterns
- Async job system for bulk operations
- Complete test coverage
- Docker deployment ready

## 📝 Implementation Checklist

- [ ] Complete project setup (Phase 1-2)
- [ ] Define and generate proto files (Phase 3)
- [ ] Implement core server (Phase 4)
- [ ] Add job system (Phase 5)
- [ ] Setup database (Phase 6)
- [ ] Implement handlers (Phase 7)
- [ ] Add tests (Phase 8)
- [ ] Configure deployment (Phase 9)

## 💡 Tips

- Follow the guides in order - each builds on the previous
- Check reference materials when stuck
- Test each phase before moving to the next
- Keep the infinityai-cataloger patterns in mind

Ready? Let's start with **[00-overview/architecture.md](00-overview/architecture.md)** →