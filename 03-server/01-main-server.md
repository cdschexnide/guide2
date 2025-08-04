# Step 1: Main Server Setup

This guide covers creating the main server entry point and initialization.

## Create server/main.go

Create the main server file that sets up both gRPC and REST servers:

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
    
    "blade-ingestion-service/database"
    "blade-ingestion-service/server/blade_server"
    "blade-ingestion-service/server/job"
    "blade-ingestion-service/server/utils"
    pb "blade-ingestion-service/generated/proto"
    
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
        log.Println("No .env file found, using environment variables")
    }
    
    // Load configuration
    config, err := utils.LoadConfig()
    if err != nil {
        log.Fatalf("Failed to load configuration: %v", err)
    }
    
    // Setup logging
    setupLogging(config)
    
    // Connect to database
    db, err := database.InitDB(config)
    if err != nil {
        log.Fatalf("Failed to initialize database: %v", err)
    }
    
    // Run migrations
    if err := database.Migrate(db); err != nil {
        log.Fatalf("Failed to run migrations: %v", err)
    }
    
    // Initialize job runner (5 concurrent workers)
    jobRunner := job.NewJobRunner(db, 5)
    defer jobRunner.Shutdown()
    
    // Create server instance
    server := blade_server.NewServer(db, config, jobRunner)
    
    // Start gRPC server in a goroutine
    grpcAddr := fmt.Sprintf("%s:%s", config.Host, config.GRPCPort)
    go func() {
        if err := startGRPCServer(grpcAddr, server); err != nil {
            log.Fatalf("Failed to start gRPC server: %v", err)
        }
    }()
    
    // Give gRPC server time to start
    time.Sleep(100 * time.Millisecond)
    
    // Start REST gateway
    restAddr := fmt.Sprintf("%s:%s", config.Host, config.RESTPort)
    go func() {
        if err := startRESTGateway(restAddr, grpcAddr, config); err != nil {
            log.Fatalf("Failed to start REST gateway: %v", err)
        }
    }()
    
    log.Printf("BLADE Ingestion Service started")
    log.Printf("gRPC server listening on %s", grpcAddr)
    log.Printf("REST server listening on %s", restAddr)
    if config.EnableSwaggerUI {
        log.Printf("Swagger UI available at http://%s/swagger-ui/", restAddr)
    }
    
    // Wait for interrupt signal
    waitForShutdown()
    
    log.Println("Shutting down servers...")
}

// startGRPCServer starts the gRPC server
func startGRPCServer(addr string, server pb.BLADEIngestionServiceServer) error {
    lis, err := net.Listen("tcp", addr)
    if err != nil {
        return fmt.Errorf("failed to listen: %w", err)
    }
    
    // Create gRPC server with interceptors
    opts := []grpc.ServerOption{
        grpc.UnaryInterceptor(loggingInterceptor),
    }
    
    grpcServer := grpc.NewServer(opts...)
    
    // Register service
    pb.RegisterBLADEIngestionServiceServer(grpcServer, server)
    
    // Register reflection service for grpcurl
    reflection.Register(grpcServer)
    
    log.Printf("Starting gRPC server on %s", addr)
    return grpcServer.Serve(lis)
}

// startRESTGateway starts the REST gateway
func startRESTGateway(restAddr, grpcAddr string, config *utils.Config) error {
    ctx := context.Background()
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    // Create gRPC client connection
    conn, err := grpc.DialContext(ctx, grpcAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        return fmt.Errorf("failed to dial gRPC server: %w", err)
    }
    defer conn.Close()
    
    // Create REST gateway mux
    gwmux := runtime.NewServeMux(
        runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{}),
    )
    
    // Register gateway handlers
    err = pb.RegisterBLADEIngestionServiceHandler(ctx, gwmux, conn)
    if err != nil {
        return fmt.Errorf("failed to register gateway: %w", err)
    }
    
    // Create HTTP mux
    mux := http.NewServeMux()
    
    // Mount gateway
    mux.Handle("/", gwmux)
    
    // Add Swagger UI if enabled
    if config.EnableSwaggerUI {
        setupSwaggerUI(mux)
    }
    
    // Add health check
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    // Setup CORS
    handler := cors.New(cors.Options{
        AllowedOrigins:   []string{"*"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"*"},
        ExposedHeaders:   []string{"*"},
        AllowCredentials: true,
    }).Handler(mux)
    
    // Create HTTP server
    srv := &http.Server{
        Addr:         restAddr,
        Handler:      handler,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    log.Printf("Starting REST gateway on %s", restAddr)
    return srv.ListenAndServe()
}

// setupSwaggerUI sets up Swagger UI endpoints
func setupSwaggerUI(mux *http.ServeMux) {
    // Serve swagger.json
    mux.HandleFunc("/swagger.json", func(w http.ResponseWriter, r *http.Request) {
        http.ServeFile(w, r, "swagger/blade_ingestion.swagger.json")
    })
    
    // Serve Swagger UI
    swaggerUI := http.StripPrefix("/swagger-ui/", http.FileServer(http.Dir("./swagger/")))
    mux.Handle("/swagger-ui/", swaggerUI)
}

// loggingInterceptor logs gRPC requests
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    
    // Call handler
    resp, err := handler(ctx, req)
    
    // Log request
    duration := time.Since(start)
    statusCode := "OK"
    if err != nil {
        statusCode = "ERROR"
    }
    
    log.Printf("gRPC: %s [%s] %v", info.FullMethod, statusCode, duration)
    
    return resp, err
}

// setupLogging configures logging based on config
func setupLogging(config *utils.Config) {
    // Set log format
    if config.LogFormat == "json" {
        // In production, you might use a structured logging library
        log.SetFlags(log.Ldate | log.Ltime | log.LUTC)
    } else {
        log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
    }
    
    // Set log level (simplified for this example)
    if config.LogLevel == "debug" {
        log.Println("Debug logging enabled")
    }
}

// waitForShutdown waits for interrupt signal
func waitForShutdown() {
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan
}
```

## Key Features Implemented

### 1. **Dual Server Setup**
- gRPC server on port 9090
- REST gateway on port 9091
- Both running concurrently

### 2. **Configuration Loading**
- Environment variables via godotenv
- Structured configuration management
- Validation at startup

### 3. **Database Integration**
- Connection initialization
- Automatic migrations
- Passed to server instance

### 4. **Job System Integration**
- Job runner initialization
- Graceful shutdown
- Passed to server for async operations

### 5. **Middleware and Interceptors**
- gRPC request logging
- CORS for REST API
- JSON marshaling options

### 6. **Swagger UI**
- Optional based on config
- Serves generated swagger.json
- Available at /swagger-ui/

### 7. **Health Checks**
- Simple /healthz endpoint
- Can be extended for readiness/liveness

### 8. **Graceful Shutdown**
- Signal handling
- Cleanup on exit
- Job runner shutdown

## Testing the Main Server

Create a simple test to verify compilation:

```bash
# Try to build
go build -o bin/blade-server ./server/main.go
```

You'll get errors about missing packages - that's expected! We'll implement them next.

## Environment Setup

Make sure your `.env` file has all required values:

```bash
# Check required environment variables
grep -E "PGHOST|PGPORT|PG_DATABASE|APP_DB_ADMIN_PASSWORD" .env
```

## Next Steps

The main.go references several packages we need to create:
1. `database` package for DB initialization
2. `blade_server.NewServer()` implementation
3. `job.NewJobRunner()` implementation

✅ Main server structure created  
➡️ Continue to [02-server-struct.md](02-server-struct.md) to implement the server struct