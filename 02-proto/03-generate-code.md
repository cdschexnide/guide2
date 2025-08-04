# Step 3: Generate Code from Proto

This guide covers generating Go code from the protocol buffer definitions.

## 1. Generate Proto Code

Run the make command to generate all code:

```bash
make proto
```

Expected output:
```
Generating proto files...
Proto generation complete!
```

## 2. Verify Generated Files

Check that all files were generated:

```bash
ls -la generated/proto/
```

You should see:
- `blade_ingestion.pb.go` - Protocol buffer messages
- `blade_ingestion_grpc.pb.go` - gRPC service interfaces
- `blade_ingestion.pb.gw.go` - REST gateway handlers

Check swagger:
```bash
ls -la swagger/
```

You should see:
- `blade_ingestion.swagger.json` - OpenAPI specification

## 3. Examine Generated Code

### A. Proto Messages (`blade_ingestion.pb.go`)

This file contains all the message structs:

```go
// Example of generated message
type DataSource struct {
    Name        string
    DisplayName string
    DataType    string
    Enabled     bool
    Config      *structpb.Struct
}

type BLADEItem struct {
    ItemId                string
    DataType              string
    Data                  *structpb.Struct
    ClassificationMarking string
    LastModified          *timestamppb.Timestamp
    Metadata              map[string]string
}
```

### B. gRPC Service (`blade_ingestion_grpc.pb.go`)

This file contains the service interfaces:

```go
// Client interface
type BLADEIngestionServiceClient interface {
    AddBLADESource(ctx context.Context, in *DataSource, opts ...grpc.CallOption) (*emptypb.Empty, error)
    QueryBLADE(ctx context.Context, in *BLADEQuery, opts ...grpc.CallOption) (*BLADEQueryResponse, error)
    StartBLADEQueryJob(ctx context.Context, in *BLADEQueryJobRequest, opts ...grpc.CallOption) (*JobResponse, error)
    // ... other methods
}

// Server interface
type BLADEIngestionServiceServer interface {
    AddBLADESource(context.Context, *DataSource) (*emptypb.Empty, error)
    QueryBLADE(context.Context, *BLADEQuery) (*BLADEQueryResponse, error)
    StartBLADEQueryJob(context.Context, *BLADEQueryJobRequest) (*JobResponse, error)
    // ... other methods
    mustEmbedUnimplementedBLADEIngestionServiceServer()
}
```

### C. REST Gateway (`blade_ingestion.pb.gw.go`)

This file contains HTTP-to-gRPC translation:

```go
// Registers HTTP handlers
func RegisterBLADEIngestionServiceHandlerServer(ctx context.Context, mux *runtime.ServeMux, server BLADEIngestionServiceServer) error

// Pattern definitions
var (
    pattern_BLADEIngestionService_AddBLADESource_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2, 0, 2, 1, 1, 0, 4, 1, 5, 2}, []string{"configure", "blade", "name"}, ""))
    pattern_BLADEIngestionService_QueryBLADE_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2, 0, 1, 0, 4, 1, 5, 1}, []string{"blade", "dataType"}, ""))
    // ... other patterns
)
```

## 4. Fix Common Generation Issues

### Issue: Missing imports
If you see errors about missing imports, ensure go.mod has:
```go
require (
    google.golang.org/grpc v1.59.0
    google.golang.org/protobuf v1.31.0
    github.com/grpc-ecosystem/grpc-gateway/v2 v2.18.0
)
```

Run:
```bash
go mod tidy
```

### Issue: Old generated files
Clean and regenerate:
```bash
make proto-clean
make proto
```

### Issue: Import path mismatch
Ensure your proto file has the correct go_package:
```protobuf
option go_package = "blade-ingestion-service/generated/proto";
```

## 5. Test Generated Code Compilation

Create a simple test file `test_generated.go`:

```go
package main

import (
    "fmt"
    pb "blade-ingestion-service/generated/proto"
)

func main() {
    // Test creating a message
    ds := &pb.DataSource{
        Name:        "test-source",
        DisplayName: "Test Source",
        DataType:    "maintenance",
        Enabled:     true,
    }
    
    fmt.Printf("Created data source: %+v\n", ds)
    
    // Test creating a query
    query := &pb.BLADEQuery{
        DataType: "maintenance",
        Limit:    10,
    }
    
    fmt.Printf("Created query: %+v\n", query)
}
```

Run it:
```bash
go run test_generated.go
```

Clean up:
```bash
rm test_generated.go
```

## 6. Examine Swagger Output

Check the generated Swagger file:

```bash
# Pretty print the swagger JSON
cat swagger/blade_ingestion.swagger.json | jq . | head -50
```

You should see:
- API title and description
- All endpoints with HTTP methods
- Request/response schemas
- Field descriptions

## 7. Update .gitignore

Since generated files can be regenerated, you might want to exclude them from git:

Add to `.gitignore`:
```
# Generated proto files (optional - some teams prefer to commit these)
/generated/**/*.pb.go
/generated/**/*.pb.gw.go

# Always ignore swagger maps
swagger/*.map
```

## Generated File Summary

| File | Purpose | When to Edit |
|------|---------|--------------|
| `*.pb.go` | Message structs and serialization | Never - regenerate from proto |
| `*_grpc.pb.go` | gRPC client/server interfaces | Never - regenerate from proto |
| `*.pb.gw.go` | REST gateway handlers | Never - regenerate from proto |
| `*.swagger.json` | API documentation | Never - regenerate from proto |

## Best Practices

1. **Never edit generated files** - Always edit the proto and regenerate
2. **Commit proto files** - Source of truth for API
3. **Consider committing generated files** - Ensures consistent builds
4. **Run generation in CI** - Verify proto changes don't break generation

## Next Steps

✅ Proto setup complete  
✅ Proto definition created  
✅ Code generation successful  
➡️ Continue to [03-server/01-main-server.md](../03-server/01-main-server.md) to implement the server