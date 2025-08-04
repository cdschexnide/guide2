# Step 1: Protocol Buffer Setup

This guide covers copying the necessary proto dependencies and setting up the proto generation environment.

## 1. Copy Google Proto Dependencies

You need to copy proto files from the infinityai-cataloger or googleapis. Since they're already in your target-structure, copy them:

```bash
# Navigate to your project
cd /Users/home/Documents/databricks-function/target-structure/ingestion-function

# Copy Google API protos
cp -r ../third_party/googleapis/google/api/annotations.proto proto/google/api/
cp -r ../third_party/googleapis/google/api/http.proto proto/google/api/
cp -r ../third_party/googleapis/google/api/field_behavior.proto proto/google/api/

# Copy Google Protobuf protos
cp -r ../third_party/googleapis/google/protobuf/empty.proto proto/google/protobuf/
cp -r ../third_party/googleapis/google/protobuf/struct.proto proto/google/protobuf/
cp -r ../third_party/googleapis/google/protobuf/any.proto proto/google/protobuf/
cp -r ../third_party/googleapis/google/protobuf/timestamp.proto proto/google/protobuf/

# Copy OpenAPI v2 protos
cp -r ../third_party/grpc-gateway/protoc-gen-openapiv2/options/annotations.proto proto/protoc-gen-openapiv2/options/
cp -r ../third_party/grpc-gateway/protoc-gen-openapiv2/options/openapiv2.proto proto/protoc-gen-openapiv2/options/
```

## 2. Verify Proto Structure

After copying, verify your proto directory structure:

```bash
find proto -name "*.proto" | sort
```

Expected output:
```
proto/blade_ingestion.proto
proto/google/api/annotations.proto
proto/google/api/field_behavior.proto
proto/google/api/http.proto
proto/google/protobuf/any.proto
proto/google/protobuf/empty.proto
proto/google/protobuf/struct.proto
proto/google/protobuf/timestamp.proto
proto/protoc-gen-openapiv2/options/annotations.proto
proto/protoc-gen-openapiv2/options/openapiv2.proto
```

## 3. Update Makefile Proto Target

Update the `Makefile` to include proper proto generation:

```makefile
# Proto generation variables
PROTO_DIR := proto
GENERATED_DIR := generated/proto
PROTO_FILES := $(PROTO_DIR)/blade_ingestion.proto

# Update the proto target
proto:
	@echo "Generating proto files..."
	@mkdir -p $(GENERATED_DIR)
	protoc -I $(PROTO_DIR) \
		-I $(PROTO_DIR)/google/api \
		-I $(PROTO_DIR)/google/protobuf \
		-I $(PROTO_DIR)/protoc-gen-openapiv2/options \
		--go_out=$(GENERATED_DIR) --go_opt=paths=source_relative \
		--go-grpc_out=$(GENERATED_DIR) --go-grpc_opt=paths=source_relative \
		--grpc-gateway_out=$(GENERATED_DIR) --grpc-gateway_opt=paths=source_relative \
		--grpc-gateway_opt=generate_unbound_methods=true \
		--openapiv2_out=swagger --openapiv2_opt=allow_merge=true \
		$(PROTO_FILES)
	@echo "Proto generation complete!"

# Add proto-clean target
proto-clean:
	@echo "Cleaning generated proto files..."
	rm -f $(GENERATED_DIR)/*.pb.go
	rm -f $(GENERATED_DIR)/*.pb.gw.go
	rm -f swagger/*.swagger.json
```

## 4. Create Swagger Directory

Create the swagger directory and add a placeholder:

```bash
mkdir -p swagger
echo "# Swagger files will be generated here" > swagger/README.md
```

## 5. Test Proto Setup

Before creating the actual proto file, let's test with a minimal proto:

Create a temporary test proto `proto/test.proto`:

```protobuf
syntax = "proto3";
package test;

option go_package = "blade-ingestion-service/generated/proto";

import "google/protobuf/empty.proto";
import "google/api/annotations.proto";

service TestService {
  rpc Test(google.protobuf.Empty) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      get: "/test"
    };
  }
}
```

Try generating:
```bash
make proto
```

If successful, you should see:
- `generated/proto/test.pb.go`
- `generated/proto/test_grpc.pb.go`
- `generated/proto/test.pb.gw.go`
- `swagger/test.swagger.json`

Clean up the test files:
```bash
rm proto/test.proto
rm generated/proto/test*.go
rm swagger/test.swagger.json
```

## 6. Install Additional Proto Tools (if needed)

If you encounter any issues:

```bash
# Install buf for better proto management (optional)
brew install bufbuild/buf/buf

# Install grpcurl for testing
brew install grpcurl

# Install protoc-gen-doc for documentation (optional)
go install github.com/pseudomuto/protoc-gen-doc/cmd/protoc-gen-doc@latest
```

## 7. Create Proto Import Helper

Create `proto/imports.proto` to document available imports:

```protobuf
// This file documents available imports for BLADE ingestion service
// DO NOT COMPILE THIS FILE - For reference only

// Google API annotations for HTTP/REST mapping
import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "google/api/http.proto";

// Google protobuf types
import "google/protobuf/any.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

// OpenAPI v2 annotations
import "protoc-gen-openapiv2/options/annotations.proto";
import "protoc-gen-openapiv2/options/openapiv2.proto";
```

## Common Issues and Solutions

### Issue: Import not found
**Solution**: Check that the file exists in the proto directory and the -I flag includes the path.

### Issue: Plugin failed with status code 1
**Solution**: Ensure protoc plugins are installed and in PATH:
```bash
which protoc-gen-go
which protoc-gen-go-grpc
which protoc-gen-grpc-gateway
which protoc-gen-openapiv2
```

### Issue: Permission denied
**Solution**: Make sure the generated directory is writable:
```bash
chmod -R 755 generated/
```

## Proto Best Practices

1. **Always use `option go_package`** to specify the Go import path
2. **Use semantic versioning** in package names if needed (e.g., `blade.v1`)
3. **Add comments** to all services, methods, and fields
4. **Use field_behavior annotations** for required/optional fields
5. **Define clear HTTP mappings** for REST endpoints

## Next Steps

✅ Proto dependencies copied  
✅ Proto generation setup complete  
➡️ Continue to [02-blade-proto.md](02-blade-proto.md) to create the main proto definition