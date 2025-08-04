# Testing Guide

This guide covers testing strategies and implementation for the BLADE ingestion service.

## Testing Structure

```
tests/
├── integration/           # Integration tests
│   ├── server_test.go    # Full server integration tests
│   ├── ingestion_test.go # Ingestion flow tests
│   └── job_test.go       # Job system tests
├── unit/                 # Unit tests
│   ├── databricks_test.go
│   └── helpers_test.go
├── mocks/                # Mock implementations
│   ├── mock_databricks.go
│   └── mock_catalog.go
└── testdata/             # Test fixtures
    └── sample_data.json
```

## 1. Create Mock Implementations

### Create tests/mocks/mock_databricks.go

```go
package mocks

import (
    "context"
    "encoding/json"
    "fmt"
    "io/ioutil"
    "path/filepath"
    "time"
)

// MockDatabricksClient mocks the Databricks client
type MockDatabricksClient struct {
    Data       map[string][]map[string]interface{}
    QueryError error
    CallCount  map[string]int
}

// NewMockDatabricksClient creates a new mock client
func NewMockDatabricksClient() *MockDatabricksClient {
    return &MockDatabricksClient{
        Data:      make(map[string][]map[string]interface{}),
        CallCount: make(map[string]int),
    }
}

// LoadTestData loads test data from JSON files
func (m *MockDatabricksClient) LoadTestData(dataType string, filename string) error {
    data, err := ioutil.ReadFile(filepath.Join("testdata", filename))
    if err != nil {
        return err
    }
    
    var items []map[string]interface{}
    if err := json.Unmarshal(data, &items); err != nil {
        return err
    }
    
    m.Data[dataType] = items
    return nil
}

// ExecuteQuery mocks query execution
func (m *MockDatabricksClient) ExecuteQuery(ctx context.Context, query string) ([]map[string]interface{}, error) {
    m.CallCount["ExecuteQuery"]++
    
    if m.QueryError != nil {
        return nil, m.QueryError
    }
    
    // Simple query parsing for testing
    if contains(query, "blade_maintenance") {
        return m.Data["maintenance"], nil
    } else if contains(query, "blade_sortie") {
        return m.Data["sortie"], nil
    }
    
    // Return empty result for unknown queries
    return []map[string]interface{}{}, nil
}

// FetchBLADEItem mocks fetching a specific item
func (m *MockDatabricksClient) FetchBLADEItem(ctx context.Context, dataType, itemID string, tableName string) (map[string]interface{}, error) {
    m.CallCount["FetchBLADEItem"]++
    
    if m.QueryError != nil {
        return nil, m.QueryError
    }
    
    items, ok := m.Data[dataType]
    if !ok {
        return nil, fmt.Errorf("data type %s not found", dataType)
    }
    
    for _, item := range items {
        if id, ok := item["item_id"].(string); ok && id == itemID {
            return item, nil
        }
    }
    
    return nil, fmt.Errorf("item not found")
}

func contains(s, substr string) bool {
    return len(s) >= len(substr) && s[:len(substr)] == substr
}
```

### Create tests/mocks/mock_catalog.go

```go
package mocks

import (
    "blade-ingestion-service/database/models"
    "fmt"
)

// MockCatalogUploader mocks the catalog uploader
type MockCatalogUploader struct {
    UploadedItems   []models.BLADEItem
    UploadError     error
    ExistingItems   map[string]bool
    CallCount       map[string]int
}

// NewMockCatalogUploader creates a new mock uploader
func NewMockCatalogUploader() *MockCatalogUploader {
    return &MockCatalogUploader{
        UploadedItems: make([]models.BLADEItem, 0),
        ExistingItems: make(map[string]bool),
        CallCount:     make(map[string]int),
    }
}

// UploadItem mocks item upload
func (m *MockCatalogUploader) UploadItem(item *models.BLADEItem) error {
    m.CallCount["UploadItem"]++
    
    if m.UploadError != nil {
        return m.UploadError
    }
    
    m.UploadedItems = append(m.UploadedItems, *item)
    m.ExistingItems[fmt.Sprintf("%s-%s", item.DataType, item.ItemID)] = true
    
    return nil
}

// CheckItemExists mocks existence check
func (m *MockCatalogUploader) CheckItemExists(dataType, itemID string) (bool, error) {
    m.CallCount["CheckItemExists"]++
    
    key := fmt.Sprintf("%s-%s", dataType, itemID)
    exists := m.ExistingItems[key]
    
    return exists, nil
}

// Reset clears the mock state
func (m *MockCatalogUploader) Reset() {
    m.UploadedItems = make([]models.BLADEItem, 0)
    m.ExistingItems = make(map[string]bool)
    m.CallCount = make(map[string]int)
    m.UploadError = nil
}
```

## 2. Create Test Data

### Create tests/testdata/sample_maintenance.json

```json
[
    {
        "item_id": "MAINT-001",
        "aircraft_tail": "AF-123",
        "aircraft_type": "F-16",
        "maintenance_type": "scheduled",
        "maintenance_code": "100HR",
        "description": "100 hour inspection",
        "priority": "HIGH",
        "technician_assigned": "John Doe",
        "base_location": "Edwards AFB",
        "work_order": "WO-2024-001",
        "classification_marking": "UNCLASSIFIED"
    },
    {
        "item_id": "MAINT-002",
        "aircraft_tail": "AF-456",
        "aircraft_type": "F-22",
        "maintenance_type": "unscheduled",
        "maintenance_code": "ENGINE-01",
        "description": "Engine malfunction",
        "priority": "CRITICAL",
        "technician_assigned": "Jane Smith",
        "base_location": "Nellis AFB",
        "work_order": "WO-2024-002",
        "classification_marking": "CONFIDENTIAL"
    }
]
```

## 3. Create Integration Tests

### Create tests/integration/server_test.go

```go
package integration

import (
    "context"
    "testing"
    "time"
    
    "blade-ingestion-service/database"
    "blade-ingestion-service/database/datasource"
    "blade-ingestion-service/server/blade_server"
    "blade-ingestion-service/server/job"
    "blade-ingestion-service/server/utils"
    "blade-ingestion-service/tests/mocks"
    pb "blade-ingestion-service/generated/proto"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "google.golang.org/protobuf/types/known/emptypb"
)

// TestServerSetup tests basic server setup
func TestServerSetup(t *testing.T) {
    // Setup test database
    db, cleanup := database.SetupTestDB(t)
    defer cleanup()
    
    // Create test config
    config := &utils.Config{
        DatabricksHost: "http://mock-databricks",
        DatabricksToken: "test-token",
        StartTime: time.Now(),
    }
    
    // Create job runner
    jobRunner := job.NewJobRunner(db, 2)
    defer jobRunner.Shutdown()
    
    // Create server
    server := blade_server.NewServer(db, config, jobRunner)
    
    assert.NotNil(t, server)
    
    // Test health check
    ctx := context.Background()
    resp, err := server.HealthCheck(ctx, &emptypb.Empty{})
    
    require.NoError(t, err)
    assert.Equal(t, "healthy", resp.Status)
}

// TestDataSourceManagement tests CRUD operations on data sources
func TestDataSourceManagement(t *testing.T) {
    db, cleanup := database.SetupTestDB(t)
    defer cleanup()
    
    config := &utils.Config{
        DatabricksHost: "http://mock-databricks",
        StartTime: time.Now(),
    }
    
    jobRunner := job.NewJobRunner(db, 2)
    defer jobRunner.Shutdown()
    
    server := blade_server.NewServer(db, config, jobRunner)
    ctx := context.Background()
    
    // Test adding a data source
    t.Run("AddDataSource", func(t *testing.T) {
        req := &pb.DataSource{
            Name:        "test-maintenance",
            DisplayName: "Test Maintenance Data",
            DataType:    "maintenance",
            Enabled:     true,
        }
        
        _, err := server.AddBLADESource(ctx, req)
        require.NoError(t, err)
        
        // Verify in database
        var ds datasource.DataSource
        err = db.Where("type_name = ?", "test-maintenance").First(&ds).Error
        require.NoError(t, err)
        assert.Equal(t, "Test Maintenance Data", ds.DisplayName)
    })
    
    // Test listing data sources
    t.Run("ListDataSources", func(t *testing.T) {
        resp, err := server.ListBLADESources(ctx, &emptypb.Empty{})
        require.NoError(t, err)
        
        // Should have default sources + our test source
        assert.GreaterOrEqual(t, len(resp.DataSources), 1)
        
        found := false
        for _, ds := range resp.DataSources {
            if ds.Name == "test-maintenance" {
                found = true
                break
            }
        }
        assert.True(t, found, "Test data source not found in list")
    })
    
    // Test removing data source
    t.Run("RemoveDataSource", func(t *testing.T) {
        req := &pb.DataSourceRequest{
            Name: "test-maintenance",
        }
        
        _, err := server.RemoveBLADESource(ctx, req)
        require.NoError(t, err)
        
        // Verify removed from database
        var count int64
        db.Model(&datasource.DataSource{}).Where("type_name = ?", "test-maintenance").Count(&count)
        assert.Equal(t, int64(0), count)
    })
}

// TestQueryEndpoints tests the query functionality
func TestQueryEndpoints(t *testing.T) {
    db, cleanup := database.SetupTestDB(t)
    defer cleanup()
    
    // Setup mock Databricks
    mockDatabricks := mocks.NewMockDatabricksClient()
    err := mockDatabricks.LoadTestData("maintenance", "sample_maintenance.json")
    require.NoError(t, err)
    
    // Create server with mock
    config := &utils.Config{
        DatabricksHost: "http://mock-databricks",
        StartTime: time.Now(),
    }
    
    jobRunner := job.NewJobRunner(db, 2)
    defer jobRunner.Shutdown()
    
    server := blade_server.NewServer(db, config, jobRunner)
    // Replace Databricks client with mock
    server.SetDatabricksClient(mockDatabricks)
    
    ctx := context.Background()
    
    // Add a data source first
    _, err = server.AddBLADESource(ctx, &pb.DataSource{
        Name:     "test-maintenance",
        DataType: "maintenance",
        Enabled:  true,
    })
    require.NoError(t, err)
    
    // Test querying BLADE data
    t.Run("QueryBLADE", func(t *testing.T) {
        req := &pb.BLADEQuery{
            DataType: "maintenance",
            Limit:    10,
        }
        
        resp, err := server.QueryBLADE(ctx, req)
        require.NoError(t, err)
        
        assert.Len(t, resp.Items, 2)
        assert.Equal(t, "MAINT-001", resp.Items[0].ItemId)
    })
    
    // Test getting specific item
    t.Run("GetBLADEItem", func(t *testing.T) {
        req := &pb.BLADEItemRequest{
            DataType: "maintenance",
            ItemId:   "MAINT-001",
        }
        
        item, err := server.GetBLADEItem(ctx, req)
        require.NoError(t, err)
        
        assert.Equal(t, "MAINT-001", item.ItemId)
        assert.Equal(t, "maintenance", item.DataType)
        assert.Equal(t, "UNCLASSIFIED", item.ClassificationMarking)
    })
}
```

### Create tests/integration/ingestion_test.go

```go
package integration

import (
    "context"
    "testing"
    
    "blade-ingestion-service/database"
    "blade-ingestion-service/database/models"
    "blade-ingestion-service/server/blade_server"
    "blade-ingestion-service/server/job"
    "blade-ingestion-service/server/utils"
    "blade-ingestion-service/tests/mocks"
    pb "blade-ingestion-service/generated/proto"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// TestIngestionFlow tests the complete ingestion flow
func TestIngestionFlow(t *testing.T) {
    db, cleanup := database.SetupTestDB(t)
    defer cleanup()
    
    // Setup mocks
    mockDatabricks := mocks.NewMockDatabricksClient()
    mockDatabricks.LoadTestData("maintenance", "sample_maintenance.json")
    
    mockCatalog := mocks.NewMockCatalogUploader()
    
    // Create server
    config := &utils.Config{
        DatabricksHost: "http://mock-databricks",
        StartTime: time.Now(),
    }
    
    jobRunner := job.NewJobRunner(db, 2)
    defer jobRunner.Shutdown()
    
    server := blade_server.NewServer(db, config, jobRunner)
    server.SetDatabricksClient(mockDatabricks)
    server.SetCatalogClient(mockCatalog)
    
    ctx := context.Background()
    
    // Add data source
    _, err := server.AddBLADESource(ctx, &pb.DataSource{
        Name:     "test-maintenance",
        DataType: "maintenance",
        Enabled:  true,
    })
    require.NoError(t, err)
    
    // Test single item ingestion
    t.Run("IngestSingleItem", func(t *testing.T) {
        req := &pb.BLADEItemRequest{
            DataType: "maintenance",
            ItemId:   "MAINT-001",
        }
        
        resp, err := server.IngestBLADEItem(ctx, req)
        require.NoError(t, err)
        
        assert.Equal(t, "success", resp.Status)
        assert.Equal(t, int32(1), resp.ItemsProcessed)
        assert.Equal(t, int32(1), resp.ItemsSucceeded)
        
        // Verify in database
        var item models.BLADEItem
        err = db.Where("item_id = ?", "MAINT-001").First(&item).Error
        require.NoError(t, err)
        assert.Equal(t, "maintenance", item.DataType)
        
        // Verify catalog upload
        assert.Len(t, mockCatalog.UploadedItems, 1)
        assert.Equal(t, "MAINT-001", mockCatalog.UploadedItems[0].ItemID)
    })
    
    // Test bulk ingestion
    t.Run("BulkIngest", func(t *testing.T) {
        mockCatalog.Reset()
        
        req := &pb.BulkIngestionRequest{
            DataType: "maintenance",
            ItemIds:  []string{"MAINT-001", "MAINT-002"},
        }
        
        resp, err := server.BulkIngestBLADE(ctx, req)
        require.NoError(t, err)
        
        assert.Equal(t, "success", resp.Status)
        assert.Equal(t, int32(2), resp.ItemsProcessed)
        assert.Equal(t, int32(2), resp.ItemsSucceeded)
        
        // Verify catalog uploads
        assert.Len(t, mockCatalog.UploadedItems, 2)
    })
    
    // Test duplicate handling
    t.Run("DuplicateHandling", func(t *testing.T) {
        // Mark item as already in catalog
        mockCatalog.ExistingItems["maintenance-MAINT-001"] = true
        
        req := &pb.BLADEItemRequest{
            DataType: "maintenance",
            ItemId:   "MAINT-001",
        }
        
        resp, err := server.IngestBLADEItem(ctx, req)
        require.NoError(t, err)
        
        assert.Equal(t, "already_ingested", resp.Status)
    })
}

// TestIngestionErrors tests error handling in ingestion
func TestIngestionErrors(t *testing.T) {
    db, cleanup := database.SetupTestDB(t)
    defer cleanup()
    
    mockDatabricks := mocks.NewMockDatabricksClient()
    mockCatalog := mocks.NewMockCatalogUploader()
    
    config := &utils.Config{
        DatabricksHost: "http://mock-databricks",
        StartTime: time.Now(),
    }
    
    jobRunner := job.NewJobRunner(db, 2)
    defer jobRunner.Shutdown()
    
    server := blade_server.NewServer(db, config, jobRunner)
    server.SetDatabricksClient(mockDatabricks)
    server.SetCatalogClient(mockCatalog)
    
    ctx := context.Background()
    
    // Test ingestion with no data source
    t.Run("NoDataSource", func(t *testing.T) {
        req := &pb.BLADEItemRequest{
            DataType: "unknown",
            ItemId:   "TEST-001",
        }
        
        _, err := server.IngestBLADEItem(ctx, req)
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "no enabled data source")
    })
    
    // Test catalog upload failure
    t.Run("CatalogUploadFailure", func(t *testing.T) {
        // Add data source
        server.AddBLADESource(ctx, &pb.DataSource{
            Name:     "test-maintenance",
            DataType: "maintenance",
            Enabled:  true,
        })
        
        // Set up mock to fail
        mockCatalog.UploadError = fmt.Errorf("catalog unavailable")
        mockDatabricks.Data["maintenance"] = []map[string]interface{}{
            {"item_id": "TEST-001", "data": "test"},
        }
        
        req := &pb.BLADEItemRequest{
            DataType: "maintenance",
            ItemId:   "TEST-001",
        }
        
        resp, err := server.IngestBLADEItem(ctx, req)
        require.NoError(t, err)
        
        assert.Equal(t, "partial_success", resp.Status)
        assert.Contains(t, resp.Errors[0], "catalog unavailable")
        
        // Item should still be in local DB
        var count int64
        db.Model(&models.BLADEItem{}).Where("item_id = ?", "TEST-001").Count(&count)
        assert.Equal(t, int64(1), count)
    })
}
```

## 4. Create Unit Tests

### Create tests/unit/databricks_test.go

```go
package unit

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "blade-ingestion-service/server/blade_server"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestDatabricksClient(t *testing.T) {
    // Create test server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        switch r.URL.Path {
        case "/api/2.0/sql/statements":
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusOK)
            w.Write([]byte(`{
                "result": {
                    "data": [["TEST-001", "maintenance", "test data"]],
                    "schema": {
                        "columns": [
                            {"name": "item_id"},
                            {"name": "data_type"},
                            {"name": "description"}
                        ]
                    }
                }
            }`))
        default:
            w.WriteHeader(http.StatusNotFound)
        }
    }))
    defer server.Close()
    
    // Create client
    client := blade_server.NewDatabricksClient(server.URL, "test-token", "test-warehouse")
    
    // Test query execution
    t.Run("ExecuteQuery", func(t *testing.T) {
        rows, err := client.ExecuteQuery(context.Background(), "SELECT * FROM test")
        require.NoError(t, err)
        
        assert.Len(t, rows, 1)
        assert.Equal(t, "TEST-001", rows[0]["item_id"])
        assert.Equal(t, "maintenance", rows[0]["data_type"])
    })
}

func TestTransformToBLADEItem(t *testing.T) {
    rawData := map[string]interface{}{
        "item_id":               "TEST-001",
        "description":           "Test item",
        "classification_marking": "SECRET",
    }
    
    item, err := blade_server.TransformToBLADEItem("maintenance", rawData)
    require.NoError(t, err)
    
    assert.Equal(t, "TEST-001", item.ItemID)
    assert.Equal(t, "maintenance", item.DataType)
    assert.Equal(t, "SECRET", item.ClassificationMarking)
    assert.NotNil(t, item.Data)
    assert.NotNil(t, item.Metadata)
}
```

## 5. Create Test Utilities

### Create tests/utils.go

```go
package tests

import (
    "context"
    "testing"
    "time"
    
    "blade-ingestion-service/server/blade_server"
    pb "blade-ingestion-service/generated/proto"
    
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/test/bufconn"
)

const bufSize = 1024 * 1024

// StartTestServer starts a gRPC server for testing
func StartTestServer(t *testing.T, server pb.BLADEIngestionServiceServer) (*grpc.ClientConn, func()) {
    lis := bufconn.Listen(bufSize)
    
    grpcServer := grpc.NewServer()
    pb.RegisterBLADEIngestionServiceServer(grpcServer, server)
    
    go func() {
        if err := grpcServer.Serve(lis); err != nil {
            t.Logf("Server exited with error: %v", err)
        }
    }()
    
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    conn, err := grpc.DialContext(ctx, "bufnet",
        grpc.WithContextDialer(func(context.Context, string) (net.Conn, error) {
            return lis.Dial()
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    
    if err != nil {
        t.Fatalf("Failed to dial bufnet: %v", err)
    }
    
    cleanup := func() {
        conn.Close()
        grpcServer.Stop()
    }
    
    return conn, cleanup
}

// CreateTestClient creates a test gRPC client
func CreateTestClient(conn *grpc.ClientConn) pb.BLADEIngestionServiceClient {
    return pb.NewBLADEIngestionServiceClient(conn)
}

// WaitForCondition waits for a condition to be true
func WaitForCondition(t *testing.T, condition func() bool, timeout time.Duration, message string) {
    deadline := time.Now().Add(timeout)
    
    for time.Now().Before(deadline) {
        if condition() {
            return
        }
        time.Sleep(100 * time.Millisecond)
    }
    
    t.Fatalf("Timeout waiting for condition: %s", message)
}
```

## 6. Running Tests

### Create Makefile test targets

Add to your Makefile:

```makefile
# Test commands
test: test-unit test-integration

test-unit:
	@echo "Running unit tests..."
	go test -v -race ./tests/unit/...

test-integration:
	@echo "Running integration tests..."
	go test -v -race -timeout 30s ./tests/integration/...

test-coverage:
	@echo "Running tests with coverage..."
	go test -v -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

test-bench:
	@echo "Running benchmarks..."
	go test -bench=. -benchmem ./...

# Run specific test
test-specific:
	go test -v -run $(TEST_NAME) ./...
```

### Example Test Commands

```bash
# Run all tests
make test

# Run unit tests only
make test-unit

# Run integration tests only
make test-integration

# Run with coverage
make test-coverage

# Run specific test
make test-specific TEST_NAME=TestIngestionFlow

# Run tests with verbose output
go test -v ./...

# Run tests in specific package
go test -v ./tests/integration/...
```

## 7. CI/CD Integration

### GitHub Actions Example

Create `.github/workflows/test.yml`:

```yaml
name: Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: blade_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Install dependencies
      run: |
        go mod download
        go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    
    - name: Run linters
      run: golangci-lint run
    
    - name: Run tests
      env:
        PGHOST: localhost
        PGPORT: 5432
        PG_DATABASE: blade_test
        APP_DB_USER: postgres
        APP_DB_ADMIN_PASSWORD: postgres
      run: |
        make test-coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out
```

## Testing Best Practices

1. **Use table-driven tests** for comprehensive coverage
2. **Mock external dependencies** (Databricks, Catalog)
3. **Test error conditions** thoroughly
4. **Use test fixtures** for consistent data
5. **Run tests in parallel** where possible
6. **Clean up resources** after tests
7. **Use meaningful test names** that describe the scenario
8. **Test timeouts** for long-running operations

## Next Steps

✅ Testing structure created  
✅ Mock implementations provided  
✅ Integration tests implemented  
✅ Unit tests implemented  
✅ CI/CD configuration provided  
➡️ Continue to [07-deployment/01-deployment-guide.md](../07-deployment/01-deployment-guide.md) for deployment