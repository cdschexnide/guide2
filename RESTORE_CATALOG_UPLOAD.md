# Restoring Catalog Upload Functionality

This guide provides step-by-step instructions to restore the catalog upload functionality that was previously removed from the minimal BLADE service.

## Overview

The catalog upload functionality allows the BLADE service to upload processed items to a catalog system. This feature was removed to simplify the initial implementation, but can be restored by following these steps.

## Components to Restore

### 1. Mock Catalog Service (Docker Compose)

**File:** `docker-compose.yml`

Uncomment and configure the mock-catalog service:

```yaml
  mock-catalog:
    image: httpd:2.4-alpine
    restart: unless-stopped
    ports:
      - "8082:80"
    networks:
      - minimal_blade_network
    volumes:
      - ./mock-catalog:/usr/local/apache2/htdocs
    command: |
      sh -c "
        mkdir -p /usr/local/apache2/htdocs &&
        echo '{\"status\":\"success\",\"message\":\"Mock catalog upload\",\"id\":\"mock-catalog-id-\$$(date +%s)\"}' > /usr/local/apache2/htdocs/upload.json &&
        httpd-foreground
      "
```

**Add dependency to minimal-blade-service:**

```yaml
  minimal-blade-service:
    # ... existing configuration ...
    depends_on:
      postgres:
        condition: service_healthy
      mock-databricks:
        condition: service_healthy
      mock-catalog:
        condition: service_started  # Add this dependency
```

### 2. Catalog Upload Implementation

**Create:** `server/blade_server/catalog_upload.go`

```go
package blade_server

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"mime/multipart"
	"net/http"
	"os"
	"strings"
	"time"

	pb "minimal-blade-ingestion/generated/proto"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// ItemData struct to keep consistent in passing file data regardless of datasource
// follows infinityai-cataloger pattern exactly
type ItemData struct {
	Data       []byte
	DataSource string
	Name       string
	URL        string
}

// CatalogResponse represents the response from catalog upload
type CatalogResponse struct {
	Status    string `json:"status"`
	Message   string `json:"message"`
	ID        string `json:"id"`
	CatalogID string `json:"catalogId,omitempty"`
}

// sendItemToCatalog uploads item data to catalog following infinityai-cataloger pattern
func (s *Server) sendItemToCatalog(itemData ItemData, jsonMap map[string]interface{}, auth string) (*pb.CatalogUploadResponse, error) {
	// Basic validation - keep the infinityai-cataloger pattern
	if itemData.Data == nil {
		e := status.Error(codes.InvalidArgument, "no file item to send")
		log.Println(e)
		return nil, e
	}

	if itemData.DataSource == "" {
		e := status.Error(codes.InvalidArgument, "dataSource should not be empty")
		log.Println(e)
		return nil, e
	}

	log.Printf("sendItemToCatalog called for item %s from source %s", 
		itemData.Name, itemData.DataSource)

	// Get catalog service URL from environment
	catalogURL := os.Getenv("CATALOG_SERVICE_URL")
	if catalogURL == "" {
		catalogURL = "http://mock-catalog:8082"
	}

	// Create multipart form data
	var requestBody bytes.Buffer
	writer := multipart.NewWriter(&requestBody)

	// Add file data
	fileWriter, err := writer.CreateFormFile("file", itemData.Name)
	if err != nil {
		log.Printf("Error creating form file: %v", err)
		return nil, status.Error(codes.Internal, "failed to prepare upload")
	}
	
	if _, err := fileWriter.Write(itemData.Data); err != nil {
		log.Printf("Error writing file data: %v", err)
		return nil, status.Error(codes.Internal, "failed to prepare file data")
	}

	// Add metadata fields
	if err := writer.WriteField("dataSource", itemData.DataSource); err != nil {
		log.Printf("Error adding dataSource field: %v", err)
		return nil, status.Error(codes.Internal, "failed to add metadata")
	}

	if err := writer.WriteField("url", itemData.URL); err != nil {
		log.Printf("Error adding url field: %v", err)
		return nil, status.Error(codes.Internal, "failed to add metadata")
	}

	// Add JSON metadata if provided
	if len(jsonMap) > 0 {
		jsonData, err := json.Marshal(jsonMap)
		if err != nil {
			log.Printf("Error marshaling JSON metadata: %v", err)
		} else {
			if err := writer.WriteField("metadata", string(jsonData)); err != nil {
				log.Printf("Error adding metadata field: %v", err)
			}
		}
	}

	writer.Close()

	// Create HTTP request
	uploadURL := fmt.Sprintf("%s/upload", catalogURL)
	req, err := http.NewRequest("POST", uploadURL, &requestBody)
	if err != nil {
		log.Printf("Error creating request: %v", err)
		return nil, status.Error(codes.Internal, "failed to create upload request")
	}

	// Set headers
	req.Header.Set("Content-Type", writer.FormDataContentType())
	
	// Add authentication if provided
	if auth != "" {
		if strings.HasPrefix(auth, "Bearer ") {
			req.Header.Set("Authorization", auth)
		} else {
			req.Header.Set("Authorization", "Bearer "+auth)
		}
	}

	// Set default auth token from environment if no auth provided
	if auth == "" {
		if token := os.Getenv("CATALOG_AUTH_TOKEN"); token != "" {
			req.Header.Set("Authorization", "Bearer "+token)
		}
	}

	// Make the request
	client := &http.Client{
		Timeout: 30 * time.Second,
	}

	log.Printf("Uploading to catalog: %s", uploadURL)
	resp, err := client.Do(req)
	if err != nil {
		log.Printf("Error uploading to catalog: %v", err)
		return nil, status.Error(codes.Internal, fmt.Sprintf("catalog upload failed: %v", err))
	}
	defer resp.Body.Close()

	// Read response
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		log.Printf("Error reading catalog response: %v", err)
		return nil, status.Error(codes.Internal, "failed to read catalog response")
	}

	log.Printf("Catalog response [%d]: %s", resp.StatusCode, string(body))

	// Handle non-2xx responses
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return nil, status.Error(codes.Internal, 
			fmt.Sprintf("catalog upload failed with status %d: %s", resp.StatusCode, string(body)))
	}

	// Parse response
	var catalogResp CatalogResponse
	if err := json.Unmarshal(body, &catalogResp); err != nil {
		log.Printf("Error parsing catalog response: %v", err)
		// Return success with basic info if we can't parse the response
		return &pb.CatalogUploadResponse{
			Status:    "success",
			Message:   "Item uploaded to catalog successfully",
			ItemId:    extractItemId(itemData.Name),
			CatalogId: "unknown",
		}, nil
	}

	// Return successful response
	return &pb.CatalogUploadResponse{
		Status:    catalogResp.Status,
		Message:   catalogResp.Message,
		ItemId:    extractItemId(itemData.Name),
		CatalogId: catalogResp.ID,
	}, nil
}

// extractItemId extracts item ID from filename, following infinityai-cataloger pattern
func extractItemId(filename string) string {
	// Remove file extension
	if idx := strings.LastIndex(filename, "."); idx != -1 {
		filename = filename[:idx]
	}
	
	// Extract ID portion (everything after last hyphen or underscore)
	if idx := strings.LastIndexAny(filename, "-_"); idx != -1 {
		return filename[idx+1:]
	}
	
	// Return the whole filename if no separator found
	return filename
}
```

### 3. Update Server Implementation

**File:** `server/blade_server/server.go`

Replace the TODO comment in `IngestBLADEItem` function:

```go
// IngestBLADEItem ingests a BLADE item to catalog
func (s *Server) IngestBLADEItem(ctx context.Context, req *pb.BladeDatum) (*pb.CatalogUploadResponse, error) {
	log.Printf("Ingesting BLADE item: %s from source: %s", req.Id, req.DataSource)

	if req.DataSource == "" {
		return nil, status.Error(codes.InvalidArgument, "dataSource is required")
	}
	if req.Id == "" {
		return nil, status.Error(codes.InvalidArgument, "id is required")
	}

	// find the data source
	var ds datasource.DataSource
	if err := s.DB.Where("type_name = ?", strings.ToLower(req.DataSource)).First(&ds).Error; err != nil {
		if err == gorm.ErrRecordNotFound {
			return nil, status.Errorf(codes.NotFound, "data source %s not found", req.DataSource)
		}
		return nil, status.Error(codes.Internal, "failed to find data source")
	}

	// Create mock data for the item
	mockData := []byte(`{"id":"` + req.Id + `","type":"blade","data":"mock BLADE data","source":"` + req.DataSource + `"}`)
	
	// follows infinityai-cataloger pattern
	itemData := ItemData{
		Data:       mockData,
		DataSource: "BLADE: " + req.DataSource,
		Name:       req.Id + ".json",
		URL:        "Retrieved from BLADE source",
	}

	// Convert protobuf Struct to map if provided
	jsonMap := make(map[string]interface{})
	if req.Json != nil {
		jsonMap = req.Json.AsMap()
	}

	// Get auth token from context or environment
	authToken := os.Getenv("CATALOG_AUTH_TOKEN")
	
	// Upload to catalog
	response, err := s.sendItemToCatalog(itemData, jsonMap, authToken)
	if err != nil {
		log.Printf("Failed to upload item to catalog: %v", err)
		return nil, err
	}

	log.Printf("Successfully ingested item %s to catalog with ID %s", req.Id, response.CatalogId)
	return response, nil
}
```

Add the required import:

```go
import (
	"context"
	"log"
	"os"          // Add this import
	"strings"

	"minimal-blade-ingestion/database/datasource"
	pb "minimal-blade-ingestion/generated/proto"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"
	"gorm.io/gorm"
)
```

### 4. Update Protocol Buffer Definitions

**File:** `proto/blade_minimal.proto`

Update the service description and response message:

```protobuf
  // Ingest a BLADE item to catalog
  rpc IngestBLADEItem(BladeDatum) returns (CatalogUploadResponse) {
    option (google.api.http) = {
      post: "/blade/ingest"
      body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Ingestion";
      summary: "Ingest BLADE item to catalog";
      description: "Fetches and ingests a BLADE item to the catalog system.";
    };
  }
```

Update the response message documentation:

```protobuf
// Response from catalog upload operation
message CatalogUploadResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Upload status (success/error)"
      example: "\"success\""
    }];
  
  string message = 2 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Human-readable status message"
      example: "\"Item uploaded successfully\""
    }];
  
  string itemId = 3 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "ID of the uploaded item"
      example: "\"test-item-123\""
    }];
  
  string catalogId = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Catalog-assigned ID for the item"
      example: "\"catalog-id-1234567890\""
    }];
}
```

### 5. Environment Configuration

**File:** `docker-compose.yml`

Update the minimal-blade-service environment variables:

```yaml
  minimal-blade-service:
    # ... existing configuration ...
    environment:
      # ... existing variables ...
      
      # Catalog configuration
      CATALOG_SERVICE_URL: http://mock-catalog:8082
      CATALOG_AUTH_TOKEN: mock-catalog-token
```

### 6. Create Mock Catalog Directory Structure

Create the mock catalog directory and upload endpoint:

```bash
mkdir -p mock-catalog
```

**Create:** `mock-catalog/upload`

This will be a simple script that the mock catalog container will serve. The httpd container will serve static files, but for a more realistic mock, you might want to create a simple upload handler.

**Alternative: Enhanced Mock Catalog Service**

For a more realistic catalog service, create a simple Go service:

**Create:** `mock-catalog-service/main.go`

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"
)

type UploadResponse struct {
	Status    string `json:"status"`
	Message   string `json:"message"`
	ID        string `json:"id"`
	CatalogID string `json:"catalogId"`
}

func uploadHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	// Parse multipart form
	if err := r.ParseMultipartForm(10 << 20); err != nil { // 10MB limit
		http.Error(w, "Failed to parse form", http.StatusBadRequest)
		return
	}

	// Log the upload details
	dataSource := r.FormValue("dataSource")
	url := r.FormValue("url")
	metadata := r.FormValue("metadata")
	
	file, header, err := r.FormFile("file")
	if err != nil {
		http.Error(w, "No file provided", http.StatusBadRequest)
		return
	}
	defer file.Close()

	log.Printf("Received upload: file=%s, dataSource=%s, url=%s, metadata=%s", 
		header.Filename, dataSource, url, metadata)

	// Generate mock catalog ID
	catalogID := fmt.Sprintf("catalog-id-%d", time.Now().Unix())

	// Return success response
	response := UploadResponse{
		Status:    "success",
		Message:   "Item uploaded to catalog successfully",
		ID:        catalogID,
		CatalogID: catalogID,
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status":"ok","service":"mock-catalog"}`))
}

func main() {
	http.HandleFunc("/upload", uploadHandler)
	http.HandleFunc("/health", healthHandler)
	
	log.Println("Mock Catalog Service starting on port 8082")
	log.Fatal(http.ListenAndServe(":8082", nil))
}
```

**Update docker-compose.yml for enhanced mock catalog:**

```yaml
  mock-catalog:
    build: ./mock-catalog-service
    restart: unless-stopped
    ports:
      - "8082:8082"
    networks:
      - minimal_blade_network
```

### 7. Update Documentation

**File:** `README.md`

Update the feature list and architecture description:

```markdown
## Features

- **Configuration Management**: Add/list/remove BLADE data sources
- **Data Ingestion**: Single item ingestion to catalog system
- **Catalog Upload**: Multipart file upload to catalog service
- **Proto-first API**: gRPC with REST gateway and Swagger UI
- **Database Integration**: PostgreSQL with GORM

## Services Available

When running with Docker Compose:
- **Minimal BLADE Service**: 
  - gRPC: localhost:9090
  - REST API: localhost:9091  
  - Swagger UI: http://localhost:9091/swagger-ui/
- **Mock Databricks**: http://localhost:8080
- **Mock Catalog**: http://localhost:8082
- **PostgreSQL**: localhost:5432

### Ingestion
- `POST /blade/ingest` - Ingest BLADE item to catalog
```

Update the architecture diagram:

```
minimal-latest-implementation/
├── proto/                    # Protocol buffer definitions
├── generated/               # Auto-generated gRPC code  
├── server/
│   ├── main.go             # Server entry point
│   ├── blade_server/       # Core handlers
│   │   ├── server.go       # Main service implementation
│   │   └── catalog_upload.go # sendItemToCatalog function
│   └── utils/              # Configuration utilities
├── database/               # Database models and connection
├── mock-catalog-service/   # Mock catalog upload service
└── swagger/                # Generated OpenAPI documentation
```

## Implementation Steps

### Step 1: Enable Mock Catalog Service

1. Uncomment the mock-catalog service in `docker-compose.yml`
2. Add the catalog dependency to minimal-blade-service
3. Update environment variables for catalog configuration

### Step 2: Implement Catalog Upload Logic

1. Create `server/blade_server/catalog_upload.go` with the provided code
2. Update `server/blade_server/server.go` to use the catalog upload functionality
3. Add the required imports

### Step 3: Update Protocol Definitions

1. Update the proto file descriptions and documentation
2. Regenerate the protocol buffer files:
   ```bash
   make proto
   ```

### Step 4: Test the Implementation

1. Start all services:
   ```bash
   docker compose up -d --build
   ```

2. Test catalog upload:
   ```bash
   # Add a data source
   curl -X GET "http://localhost:9091/configure/blade/test-source"
   
   # Upload an item to catalog
   curl -X POST "http://localhost:9091/blade/ingest" \
     -H "Content-Type: application/json" \
     -d '{"dataSource":"test-source","id":"test-item-123","getFile":false}'
   ```

3. Check catalog service health:
   ```bash
   curl http://localhost:8082/health
   ```

4. Verify logs show successful catalog upload:
   ```bash
   docker compose logs minimal-blade-service
   docker compose logs mock-catalog
   ```

## Security Considerations

- The mock catalog service accepts any authentication token for development
- In production, implement proper authentication and authorization
- Validate file types and sizes before upload
- Implement rate limiting and request size limits
- Use HTTPS for all communications

## Troubleshooting

### Common Issues

1. **Catalog service not responding**
   - Check that mock-catalog service is running: `docker compose ps`
   - Verify network connectivity between services
   - Check service logs: `docker compose logs mock-catalog`

2. **Upload timeouts**
   - Increase client timeout in `catalog_upload.go`
   - Check file sizes and network conditions

3. **Authentication errors**
   - Verify `CATALOG_AUTH_TOKEN` environment variable is set
   - Check that the mock catalog service accepts the token format

This restoration guide provides complete instructions for re-implementing the catalog upload functionality that was previously removed from the system.
