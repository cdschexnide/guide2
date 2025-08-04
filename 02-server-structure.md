# Server Structure Implementation (server/blade_server/server.go)

This guide explains how to implement the main server structure and basic setup after generating protobuf code.

## Overview

The `server/blade_server/server.go` file contains the core server struct and basic service implementation that satisfies the generated protobuf service interface.

## Prerequisites

- Generated protobuf code with `BLADEMinimalServiceServer` interface
- Database package implemented
- Understanding of gRPC service patterns

## Server Structure Implementation

### File: `server/blade_server/server.go`

```go
package blade_server

import (
	"context"
	"log"
	"strings"

	"minimal-blade-ingestion/database/datasource"
	pb "minimal-blade-ingestion/generated/proto"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"
	"gorm.io/gorm"
)

// Server implements the BLADEMinimalServiceServer interface
// This struct holds all dependencies and state needed for the service
type Server struct {
	// Embed the unimplemented server to ensure forward compatibility
	pb.UnimplementedBLADEMinimalServiceServer
	
	// Database connection for data persistence
	DB *gorm.DB
	
	// Add other dependencies here as needed:
	// - External service clients
	// - Configuration
	// - Caches
	// - Metrics collectors
}

// NewServer creates a new instance of the BLADE server
// This constructor initializes all dependencies and returns a ready-to-use server
func NewServer(db *gorm.DB) *Server {
	if db == nil {
		log.Fatal("Database connection is required")
	}

	server := &Server{
		DB: db,
	}

	log.Println("BLADE server initialized successfully")
	return server
}

// ============= Server Health and Info =============

// GetServerInfo returns basic information about the server
// This is useful for debugging and service discovery
func (s *Server) GetServerInfo(ctx context.Context, req *emptypb.Empty) (*pb.ServerInfoResponse, error) {
	log.Println("Server info requested")

	return &pb.ServerInfoResponse{
		ServiceName:    "BLADE Minimal Service",
		Version:        "1.0.0",
		Status:         "running",
		Description:    "Minimal BLADE data ingestion and processing service",
		DatabaseStatus: s.getDatabaseStatus(),
	}, nil
}

// getDatabaseStatus checks if database connection is healthy
func (s *Server) getDatabaseStatus() string {
	sqlDB, err := s.DB.DB()
	if err != nil {
		return "error"
	}

	if err := sqlDB.Ping(); err != nil {
		return "disconnected"
	}

	return "connected"
}

// ============= Configuration Management =============

// AddBLADESource adds a new BLADE data source configuration
func (s *Server) AddBLADESource(ctx context.Context, req *pb.DataSource) (*emptypb.Empty, error) {
	log.Printf("Adding BLADE source: %s", req.Name)

	// Input validation
	if req.Name == "" {
		return nil, status.Error(codes.InvalidArgument, "name is required")
	}

	// Create data source model
	newDataSource := &datasource.DataSource{
		Name:            req.Name,
		DisplayName:     req.DisplayName,
		TypeName:        strings.ToLower(req.Name),
		URI:             "/blade/" + strings.ToLower(req.Name),
		GetFile:         req.GetFile,
		IntegrationName: req.IntegrationName,
		Description:     "BLADE data source: " + req.DisplayName,
	}

	// Set default integration name if not provided
	if newDataSource.IntegrationName == "" {
		newDataSource.IntegrationName = "BLADE"
	}

	// Save to database
	if err := s.DB.Create(newDataSource).Error; err != nil {
		// Handle specific database errors
		var code codes.Code
		if strings.Contains(err.Error(), "duplicate") || strings.Contains(err.Error(), "already exists") {
			code = codes.AlreadyExists
		} else {
			code = codes.Internal
		}
		
		log.Printf("Error creating data source: %v", err)
		return nil, status.Error(code, "failed to create data source")
	}

	log.Printf("Successfully added data source: %s", req.Name)
	return &emptypb.Empty{}, nil
}

// ListBLADESources returns all configured BLADE data sources
func (s *Server) ListBLADESources(ctx context.Context, req *emptypb.Empty) (*pb.DataSourceList, error) {
	log.Println("Listing BLADE sources")

	var sources []datasource.DataSource
	if err := s.DB.Find(&sources).Error; err != nil {
		log.Printf("Error listing data sources: %v", err)
		return nil, status.Error(codes.Internal, "failed to list data sources")
	}

	// Convert database models to protobuf messages
	pbSources := make([]*pb.DataSource, len(sources))
	for i, src := range sources {
		pbSources[i] = &pb.DataSource{
			Name:            src.Name,
			DisplayName:     src.DisplayName,
			GetFile:         src.GetFile,
			IntegrationName: src.IntegrationName,
		}
	}

	return &pb.DataSourceList{
		DataSources: pbSources,
	}, nil
}

// RemoveBLADESource removes a configured BLADE data source
func (s *Server) RemoveBLADESource(ctx context.Context, req *pb.DataSource) (*emptypb.Empty, error) {
	log.Printf("Removing BLADE source: %s", req.Name)

	if req.Name == "" {
		return nil, status.Error(codes.InvalidArgument, "name is required")
	}

	// Delete from database
	result := s.DB.Where("type_name = ?", strings.ToLower(req.Name)).Delete(&datasource.DataSource{})
	if result.Error != nil {
		log.Printf("Error removing data source: %v", result.Error)
		return nil, status.Error(codes.Internal, "failed to remove data source")
	}

	// Check if any records were actually deleted
	if result.RowsAffected == 0 {
		return nil, status.Errorf(codes.NotFound, "data source %s not found", req.Name)
	}

	log.Printf("Successfully removed data source: %s", req.Name)
	return &emptypb.Empty{}, nil
}

// ============= Validation Helpers =============

// validateDataSource performs common validation on DataSource requests
func (s *Server) validateDataSource(req *pb.DataSource) error {
	if req.Name == "" {
		return status.Error(codes.InvalidArgument, "name is required")
	}

	// Check for invalid characters in name
	if strings.ContainsAny(req.Name, "!@#$%^&*()+=[]{}|;:'\",.<>?/\\`~") {
		return status.Error(codes.InvalidArgument, "name contains invalid characters")
	}

	// Length validation
	if len(req.Name) > 100 {
		return status.Error(codes.InvalidArgument, "name is too long (max 100 characters)")
	}

	if len(req.DisplayName) > 200 {
		return status.Error(codes.InvalidArgument, "displayName is too long (max 200 characters)")
	}

	return nil
}

// ============= Database Helpers =============

// findDataSourceByName finds a data source by name
func (s *Server) findDataSourceByName(name string) (*datasource.DataSource, error) {
	var ds datasource.DataSource
	if err := s.DB.Where("type_name = ?", strings.ToLower(name)).First(&ds).Error; err != nil {
		if err == gorm.ErrRecordNotFound {
			return nil, status.Errorf(codes.NotFound, "data source %s not found", name)
		}
		return nil, status.Error(codes.Internal, "failed to find data source")
	}
	return &ds, nil
}

// dataSourceExists checks if a data source with the given name already exists
func (s *Server) dataSourceExists(name string) (bool, error) {
	var count int64
	if err := s.DB.Model(&datasource.DataSource{}).
		Where("type_name = ?", strings.ToLower(name)).
		Count(&count).Error; err != nil {
		return false, err
	}
	return count > 0, nil
}

// ============= Error Handling Helpers =============

// handleDatabaseError converts database errors to appropriate gRPC status codes
func (s *Server) handleDatabaseError(err error, operation string) error {
	log.Printf("Database error during %s: %v", operation, err)

	// Map common database errors to gRPC status codes
	switch {
	case strings.Contains(err.Error(), "duplicate"):
		return status.Error(codes.AlreadyExists, "resource already exists")
	case strings.Contains(err.Error(), "not found"):
		return status.Error(codes.NotFound, "resource not found")
	case strings.Contains(err.Error(), "connection"):
		return status.Error(codes.Unavailable, "database connection error")
	case strings.Contains(err.Error(), "timeout"):
		return status.Error(codes.DeadlineExceeded, "database operation timeout")
	default:
		return status.Error(codes.Internal, "internal database error")
	}
}

// ============= Logging Helpers =============

// logRequest logs incoming requests with context information
func (s *Server) logRequest(method string, params ...interface{}) {
	log.Printf("[%s] %v", method, params)
}

// logResponse logs responses with timing information
func (s *Server) logResponse(method string, duration time.Duration, err error) {
	if err != nil {
		log.Printf("[%s] ERROR after %v: %v", method, duration, err)
	} else {
		log.Printf("[%s] SUCCESS after %v", method, duration)
	}
}
```

## Key Design Patterns

### 1. Server Struct Design

```go
type Server struct {
    pb.UnimplementedBLADEMinimalServiceServer  // Forward compatibility
    DB *gorm.DB                                // Core dependency
    // Add more dependencies as needed
}
```

**Benefits:**
- Embeds unimplemented server for forward compatibility
- Clear dependency injection through constructor
- Single source of truth for server state

### 2. Constructor Pattern

```go
func NewServer(db *gorm.DB) *Server {
    if db == nil {
        log.Fatal("Database connection is required")
    }
    // Initialize and return server
}
```

**Benefits:**
- Validates required dependencies
- Central initialization logic
- Clear interface for server creation

### 3. Input Validation

```go
if req.Name == "" {
    return nil, status.Error(codes.InvalidArgument, "name is required")
}
```

**Benefits:**
- Early validation prevents invalid state
- Consistent error responses
- Proper gRPC status codes

### 4. Error Handling Strategy

```go
func (s *Server) handleDatabaseError(err error, operation string) error {
    // Map database errors to appropriate gRPC codes
}
```

**Benefits:**
- Consistent error mapping
- Proper error logging
- Client-friendly error messages

### 5. Helper Methods

The implementation includes several categories of helpers:

- **Validation Helpers**: Input validation logic
- **Database Helpers**: Common database operations
- **Error Handling Helpers**: Error mapping and logging
- **Logging Helpers**: Structured logging

## Required Imports

```go
import (
    "context"
    "log"
    "strings"
    "time"

    "minimal-blade-ingestion/database/datasource"
    pb "minimal-blade-ingestion/generated/proto"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/emptypb"
    "gorm.io/gorm"
)
```

## Error Handling Best Practices

### 1. Use Appropriate gRPC Status Codes

```go
// Not Found
return nil, status.Error(codes.NotFound, "resource not found")

// Invalid Input
return nil, status.Error(codes.InvalidArgument, "name is required")

// Already Exists
return nil, status.Error(codes.AlreadyExists, "resource already exists")

// Internal Error
return nil, status.Error(codes.Internal, "internal server error")
```

### 2. Log Errors Appropriately

```go
log.Printf("Error creating data source: %v", err)  // Log the technical details
return nil, status.Error(code, "failed to create data source")  // Return user-friendly message
```

### 3. Validate Input Early

```go
if req.Name == "" {
    return nil, status.Error(codes.InvalidArgument, "name is required")
}
// Continue with business logic only after validation passes
```

## Testing Considerations

### 1. Unit Test Structure

```go
func TestServer_AddBLADESource(t *testing.T) {
    // Setup mock database
    // Create server instance
    // Test various scenarios
}
```

### 2. Test Cases to Cover

- Valid input scenarios
- Invalid input validation
- Database error scenarios
- Edge cases (empty lists, duplicates)

### 3. Mock Dependencies

Use interfaces and dependency injection to enable easy testing:

```go
type DatabaseInterface interface {
    Create(interface{}) error
    Find(interface{}) error
    // etc.
}
```

## Integration with Generated Code

### 1. Interface Compliance

The server must implement all methods from the generated interface:

```go
type BLADEMinimalServiceServer interface {
    AddBLADESource(context.Context, *DataSource) (*emptypb.Empty, error)
    ListBLADESources(context.Context, *emptypb.Empty) (*DataSourceList, error)
    RemoveBLADESource(context.Context, *DataSource) (*emptypb.Empty, error)
    // Additional methods...
}
```

### 2. Message Type Usage

Convert between database models and protobuf messages:

```go
// Database model to protobuf
pbSource := &pb.DataSource{
    Name:            src.Name,
    DisplayName:     src.DisplayName,
    GetFile:         src.GetFile,
    IntegrationName: src.IntegrationName,
}

// Protobuf to database model
dbSource := &datasource.DataSource{
    Name:        req.Name,
    DisplayName: req.DisplayName,
    // etc.
}
```

## Next Steps

1. Implement remaining service methods (ingestion endpoints)
2. Add proper logging and metrics
3. Implement comprehensive input validation
4. Add authentication/authorization if needed
5. Create unit tests for all methods
6. Add integration tests

This server structure provides a solid foundation for implementing the BLADE service with proper error handling, validation, and maintainability.
