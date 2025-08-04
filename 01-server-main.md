# Server Main Implementation (server/main.go)

This guide explains how to implement the main server entry point after generating protobuf code.

## Overview

The `server/main.go` file serves as the entry point for the BLADE service, setting up both gRPC and HTTP REST gateway servers with proper middleware, database connections, and graceful shutdown.

## Prerequisites

- Generated protobuf code in `generated/proto/`
- Database package implemented
- Server package structure created

## Complete Implementation

### File: `server/main.go`

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"minimal-blade-ingestion/database"
	"minimal-blade-ingestion/server/blade_server"
	pb "minimal-blade-ingestion/generated/proto"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"github.com/joho/godotenv"
	"github.com/rs/cors"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/reflection"
)

func main() {
	// Load environment variables
	if err := godotenv.Load(); err != nil {
		log.Printf("Warning: Could not load .env file: %v", err)
	}

	// Initialize database
	db, err := database.InitializeDatabase()
	if err != nil {
		log.Fatalf("Failed to initialize database: %v", err)
	}
	log.Println("Database connection established successfully")

	// Create server instance
	server := blade_server.NewServer(db)

	// Start gRPC server in a goroutine
	grpcPort := getEnv("GRPC_PORT", "9090")
	go startGRPCServer(server, grpcPort)

	// Start HTTP REST gateway server
	restPort := getEnv("REST_PORT", "9091")
	startHTTPServer(server, grpcPort, restPort)
}

func startGRPCServer(server *blade_server.Server, port string) {
	lis, err := net.Listen("tcp", ":"+port)
	if err != nil {
		log.Fatalf("Failed to listen on port %s: %v", port, err)
	}

	grpcServer := grpc.NewServer(
		grpc.UnaryInterceptor(loggingInterceptor),
		grpc.MaxRecvMsgSize(1024*1024*50), // 50MB max message size
	)

	pb.RegisterBLADEMinimalServiceServer(grpcServer, server)
	
	// Enable gRPC reflection for development
	reflection.Register(grpcServer)

	log.Printf("gRPC server starting on port %s", port)
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("Failed to serve gRPC server: %v", err)
	}
}

func startHTTPServer(server *blade_server.Server, grpcPort, restPort string) {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	// Create gRPC-Gateway mux
	mux := runtime.NewServeMux(
		runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{
			MarshalOptions: protojson.MarshalOptions{
				UseProtoNames:   true,
				EmitUnpopulated: false,
			},
		}),
	)

	opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	
	// Register gRPC-Gateway handlers
	err := pb.RegisterBLADEMinimalServiceHandlerFromEndpoint(
		ctx, mux, "localhost:"+grpcPort, opts,
	)
	if err != nil {
		log.Fatalf("Failed to register gateway: %v", err)
	}

	// Create HTTP router
	router := http.NewServeMux()
	
	// Add API routes
	router.Handle("/", mux)
	
	// Add health check endpoint
	router.HandleFunc("/healthz", healthCheckHandler)
	
	// Serve Swagger UI if enabled
	if getEnv("ENABLE_SWAGGER_UI", "true") == "true" {
		router.Handle("/swagger-ui/", http.StripPrefix("/swagger-ui/", http.FileServer(http.Dir("./swagger/"))))
		log.Println("Swagger UI available at: http://localhost:" + restPort + "/swagger-ui/")
	}

	// Configure CORS
	c := cors.New(cors.Options{
		AllowedOrigins: []string{"*"},
		AllowedMethods: []string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"},
		AllowedHeaders: []string{"*"},
		ExposedHeaders: []string{"*"},
	})

	handler := c.Handler(router)

	// Create HTTP server
	httpServer := &http.Server{
		Addr:         ":" + restPort,
		Handler:      handler,
		ReadTimeout:  30 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start server with graceful shutdown
	go func() {
		log.Printf("HTTP REST server starting on port %s", restPort)
		if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Failed to start HTTP server: %v", err)
		}
	}()

	// Wait for interrupt signal for graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Println("Shutting down servers...")

	// Graceful shutdown with timeout
	ctx, cancel = context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := httpServer.Shutdown(ctx); err != nil {
		log.Printf("HTTP server forced to shutdown: %v", err)
	}

	log.Println("Servers shutdown complete")
}

// loggingInterceptor logs gRPC calls
func loggingInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()
	
	resp, err := handler(ctx, req)
	
	duration := time.Since(start)
	status := "OK"
	if err != nil {
		status = "ERROR"
	}
	
	log.Printf("gRPC: %s [%s] %v", info.FullMethod, status, duration)
	
	if err != nil {
		log.Printf("gRPC Error: %v", err)
	}
	
	return resp, err
}

// healthCheckHandler provides a simple health check endpoint
func healthCheckHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status":"healthy","service":"minimal-blade-service","timestamp":"` + time.Now().UTC().Format(time.RFC3339) + `"}`))
}

// getEnv gets environment variable with fallback default
func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}
```

## Key Components Explained

### 1. Environment Loading

```go
if err := godotenv.Load(); err != nil {
    log.Printf("Warning: Could not load .env file: %v", err)
}
```

- Loads environment variables from `.env` file
- Non-fatal if file doesn't exist (allows for container deployment)

### 2. Database Initialization

```go
db, err := database.InitializeDatabase()
if err != nil {
    log.Fatalf("Failed to initialize database: %v", err)
}
```

- Initializes database connection and runs migrations
- Fatal error if database connection fails

### 3. Dual Server Setup

The implementation runs both gRPC and HTTP REST servers:

- **gRPC Server**: Direct protobuf communication on port 9090
- **HTTP REST Gateway**: JSON REST API on port 9091

### 4. gRPC Server Configuration

```go
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
    grpc.MaxRecvMsgSize(1024*1024*50), // 50MB max message size
)
```

- Logging interceptor for request/response logging
- Large message size support for file uploads
- gRPC reflection enabled for development tools

### 5. HTTP Gateway Configuration

```go
mux := runtime.NewServeMux(
    runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{
        MarshalOptions: protojson.MarshalOptions{
            UseProtoNames:   true,
            EmitUnpopulated: false,
        },
    }),
)
```

- JSON marshaling configuration
- Proto field names in JSON responses
- Excludes empty fields from responses

### 6. CORS Configuration

```go
c := cors.New(cors.Options{
    AllowedOrigins: []string{"*"},
    AllowedMethods: []string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"},
    AllowedHeaders: []string{"*"},
    ExposedHeaders: []string{"*"},
})
```

- Allows cross-origin requests for web development
- Configure appropriately for production

### 7. Graceful Shutdown

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

// Shutdown with 30-second timeout
ctx, cancel = context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

- Handles SIGINT and SIGTERM signals
- Graceful shutdown with timeout
- Completes in-flight requests before shutdown

## Required Dependencies

Add these to your `go.mod`:

```go
require (
    github.com/grpc-ecosystem/grpc-gateway/v2 v2.27.1
    github.com/joho/godotenv v1.4.0
    github.com/rs/cors v1.10.1
    google.golang.org/grpc v1.74.2
    google.golang.org/protobuf v1.36.6
)
```

## Environment Variables

The server supports these environment variables:

```bash
# Server ports
GRPC_PORT=9090
REST_PORT=9091

# Database configuration (handled by database package)
PGHOST=localhost
PGPORT=5432
PG_DATABASE=blade_db
PGUSER=blade_user
PGPASSWORD=blade_password

# Feature flags
ENABLE_SWAGGER_UI=true

# Logging
LOG_LEVEL=info
```

## Testing the Server

After implementation, test both servers:

```bash
# Test health check
curl http://localhost:9091/healthz

# Test gRPC reflection (if grpcurl is installed)
grpcurl -plaintext localhost:9090 list

# Test REST API endpoints
curl http://localhost:9091/configure/blade/sources
```

## Development vs Production

### Development Features

- gRPC reflection enabled
- Detailed logging
- Swagger UI served
- CORS allows all origins

### Production Considerations

- Disable gRPC reflection
- Configure specific CORS origins
- Use structured logging
- Add metrics and monitoring
- Use TLS/SSL certificates
- Configure proper timeouts

## Next Steps

1. Implement `database.InitializeDatabase()` function
2. Create `blade_server.NewServer()` constructor
3. Implement server methods for protobuf services
4. Add proper error handling and validation
5. Configure production-ready settings

This main.go implementation provides a robust foundation for the BLADE service with proper separation of concerns, error handling, and production readiness.
