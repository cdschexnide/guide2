# Step 2: Server Struct Implementation

This guide covers implementing the main server struct and its core methods.

## Create server/blade_server/server.go

Create the main server implementation that fulfills the gRPC service interface:

```go
package blade_server

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "strings"
    "time"
    
    "blade-ingestion-service/database/datasource"
    "blade-ingestion-service/database/models"
    "blade-ingestion-service/server/job"
    "blade-ingestion-service/server/utils"
    pb "blade-ingestion-service/generated/proto"
    
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/emptypb"
    "google.golang.org/protobuf/types/known/structpb"
    "google.golang.org/protobuf/types/known/timestamppb"
    "gorm.io/gorm"
)

// Server implements the BLADEIngestionServiceServer interface
type Server struct {
    pb.UnimplementedBLADEIngestionServiceServer
    
    db            *gorm.DB
    config        *utils.Config
    jobRunner     *job.JobRunner
    databricks    *DatabricksClient
    catalogClient *CatalogUploader
}

// NewServer creates a new BLADE ingestion server
func NewServer(db *gorm.DB, config *utils.Config, jobRunner *job.JobRunner) *Server {
    // Initialize Databricks client
    databricksClient := NewDatabricksClient(
        config.DatabricksHost,
        config.DatabricksToken,
        config.DatabricksWarehouseID,
    )
    
    // Initialize catalog uploader
    catalogClient := NewCatalogUploader(
        config.CatalogServiceURL,
        config.CatalogAuthToken,
    )
    
    return &Server{
        db:            db,
        config:        config,
        jobRunner:     jobRunner,
        databricks:    databricksClient,
        catalogClient: catalogClient,
    }
}

// ============= Configuration Endpoints =============

// AddBLADESource adds a new BLADE data source
func (s *Server) AddBLADESource(ctx context.Context, req *pb.DataSource) (*emptypb.Empty, error) {
    log.Printf("Adding BLADE source: %s", req.Name)
    
    // Validate input
    if req.Name == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    
    if req.DataType == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType is required")
    }
    
    // Check if source already exists
    var existing datasource.DataSource
    if err := s.db.Where("type_name = ?", req.Name).First(&existing).Error; err == nil {
        return nil, status.Errorf(codes.AlreadyExists, "data source %s already exists", req.Name)
    }
    
    // Extract config from proto Struct
    configMap := make(map[string]interface{})
    if req.Config != nil {
        configMap = req.Config.AsMap()
    }
    
    // Create new data source
    ds := &datasource.DataSource{
        TypeName:    req.Name,
        DisplayName: req.DisplayName,
        DataType:    req.DataType,
        Enabled:     req.Enabled,
        
        // Extract Databricks specific config
        WarehouseID: getStringFromMap(configMap, "warehouse_id", s.config.DatabricksWarehouseID),
        CatalogName: getStringFromMap(configMap, "catalog", "hive_metastore"),
        SchemaName:  getStringFromMap(configMap, "schema", "default"),
        TableName:   getStringFromMap(configMap, "table", fmt.Sprintf("blade_%s", req.DataType)),
    }
    
    // Set parameters
    if err := ds.SetParameters(configMap); err != nil {
        return nil, status.Error(codes.Internal, "failed to set parameters")
    }
    
    // Save to database
    if err := s.db.Create(ds).Error; err != nil {
        log.Printf("Error creating data source: %v", err)
        return nil, status.Error(codes.Internal, "failed to create data source")
    }
    
    log.Printf("Successfully added data source: %s", req.Name)
    return &emptypb.Empty{}, nil
}

// ListBLADESources lists all configured BLADE data sources
func (s *Server) ListBLADESources(ctx context.Context, req *emptypb.Empty) (*pb.DataSourceList, error) {
    log.Println("Listing BLADE sources")
    
    var sources []datasource.DataSource
    if err := s.db.Find(&sources).Error; err != nil {
        return nil, status.Error(codes.Internal, "failed to list data sources")
    }
    
    // Convert to proto format
    pbSources := make([]*pb.DataSource, len(sources))
    for i, src := range sources {
        // Get parameters
        params, _ := src.GetParameters()
        
        // Convert to proto Struct
        configStruct, err := structpb.NewStruct(params)
        if err != nil {
            log.Printf("Error converting config to struct: %v", err)
            configStruct = &structpb.Struct{}
        }
        
        pbSources[i] = &pb.DataSource{
            Name:        src.TypeName,
            DisplayName: src.DisplayName,
            DataType:    src.DataType,
            Enabled:     src.Enabled,
            Config:      configStruct,
        }
    }
    
    return &pb.DataSourceList{
        DataSources: pbSources,
    }, nil
}

// RemoveBLADESource removes a BLADE data source
func (s *Server) RemoveBLADESource(ctx context.Context, req *pb.DataSourceRequest) (*emptypb.Empty, error) {
    log.Printf("Removing BLADE source: %s", req.Name)
    
    if req.Name == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    
    // Find and delete the source
    result := s.db.Where("type_name = ?", req.Name).Delete(&datasource.DataSource{})
    if result.Error != nil {
        return nil, status.Error(codes.Internal, "failed to remove data source")
    }
    
    if result.RowsAffected == 0 {
        return nil, status.Errorf(codes.NotFound, "data source %s not found", req.Name)
    }
    
    log.Printf("Successfully removed data source: %s", req.Name)
    return &emptypb.Empty{}, nil
}

// ============= Query Endpoints =============

// QueryBLADE queries BLADE data by type
func (s *Server) QueryBLADE(ctx context.Context, req *pb.BLADEQuery) (*pb.BLADEQueryResponse, error) {
    log.Printf("Querying BLADE data - type: %s, limit: %d", req.DataType, req.Limit)
    
    // Validate input
    if req.DataType == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType is required")
    }
    
    // Get data source configuration
    var ds datasource.DataSource
    if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).First(&ds).Error; err != nil {
        return nil, status.Errorf(codes.NotFound, "no enabled data source found for type %s", req.DataType)
    }
    
    // Build query
    query := fmt.Sprintf("SELECT * FROM %s", ds.GetFullTableName())
    
    // Add filter if provided
    if req.Filter != "" {
        query += fmt.Sprintf(" WHERE %s", req.Filter)
    }
    
    // Add order by if provided
    if req.OrderBy != "" {
        query += fmt.Sprintf(" ORDER BY %s", req.OrderBy)
    }
    
    // Add limit and offset
    limit := req.Limit
    if limit <= 0 {
        limit = 100
    }
    query += fmt.Sprintf(" LIMIT %d", limit)
    
    if req.Offset > 0 {
        query += fmt.Sprintf(" OFFSET %d", req.Offset)
    }
    
    // Execute query
    rows, err := s.databricks.ExecuteQuery(ctx, query)
    if err != nil {
        log.Printf("Error executing query: %v", err)
        return nil, status.Error(codes.Internal, "failed to execute query")
    }
    
    // Convert to proto format
    items := make([]*pb.BLADEItem, len(rows))
    for i, row := range rows {
        item, err := s.convertRowToBLADEItem(req.DataType, row)
        if err != nil {
            log.Printf("Error converting row: %v", err)
            continue
        }
        items[i] = item
    }
    
    // Get total count (simplified - in production use COUNT query)
    totalCount := int32(len(items))
    if int32(len(items)) == limit {
        totalCount = -1 // Indicates there might be more
    }
    
    return &pb.BLADEQueryResponse{
        Items:      items,
        TotalCount: totalCount,
    }, nil
}

// GetBLADEItem gets a specific BLADE item
func (s *Server) GetBLADEItem(ctx context.Context, req *pb.BLADEItemRequest) (*pb.BLADEItem, error) {
    log.Printf("Getting BLADE item - type: %s, id: %s", req.DataType, req.ItemId)
    
    // Validate input
    if req.DataType == "" || req.ItemId == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType and itemId are required")
    }
    
    // Get data source configuration
    var ds datasource.DataSource
    if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).First(&ds).Error; err != nil {
        return nil, status.Errorf(codes.NotFound, "no enabled data source found for type %s", req.DataType)
    }
    
    // Fetch item from Databricks
    row, err := s.databricks.FetchBLADEItem(ctx, req.DataType, req.ItemId, ds.GetFullTableName())
    if err != nil {
        if strings.Contains(err.Error(), "not found") {
            return nil, status.Errorf(codes.NotFound, "item %s not found", req.ItemId)
        }
        return nil, status.Error(codes.Internal, "failed to fetch item")
    }
    
    // Convert to proto format
    return s.convertRowToBLADEItem(req.DataType, row)
}

// ============= Helper Methods =============

// convertRowToBLADEItem converts a database row to proto BLADEItem
func (s *Server) convertRowToBLADEItem(dataType string, row map[string]interface{}) (*pb.BLADEItem, error) {
    // Extract item ID
    itemID := ""
    if id, ok := row["item_id"].(string); ok {
        itemID = id
    }
    
    // Extract classification
    classification := models.GetDefaultClassification(models.BLADEItemType(dataType))
    if class, ok := row["classification_marking"].(string); ok && class != "" {
        classification = class
    }
    
    // Extract timestamp
    var lastModified *timestamppb.Timestamp
    if ts, ok := row["last_modified"].(time.Time); ok {
        lastModified = timestamppb.New(ts)
    } else if tsStr, ok := row["last_modified"].(string); ok {
        if t, err := time.Parse(time.RFC3339, tsStr); err == nil {
            lastModified = timestamppb.New(t)
        }
    }
    
    // Convert row data to Struct
    dataStruct, err := structpb.NewStruct(row)
    if err != nil {
        return nil, fmt.Errorf("failed to convert data to struct: %w", err)
    }
    
    // Extract metadata
    metadata := make(map[string]string)
    if meta, ok := row["metadata"].(map[string]interface{}); ok {
        for k, v := range meta {
            if strVal, ok := v.(string); ok {
                metadata[k] = strVal
            }
        }
    }
    
    return &pb.BLADEItem{
        ItemId:                itemID,
        DataType:              dataType,
        Data:                  dataStruct,
        ClassificationMarking: classification,
        LastModified:          lastModified,
        Metadata:              metadata,
    }, nil
}

// getStringFromMap safely gets a string value from a map
func getStringFromMap(m map[string]interface{}, key string, defaultValue string) string {
    if val, ok := m[key].(string); ok && val != "" {
        return val
    }
    return defaultValue
}

// ============= Health Check =============

// HealthCheck returns the service health status
func (s *Server) HealthCheck(ctx context.Context, req *emptypb.Empty) (*pb.HealthResponse, error) {
    // Check database connection
    dbStatus := "healthy"
    var result int
    if err := s.db.Raw("SELECT 1").Scan(&result).Error; err != nil {
        dbStatus = "unhealthy: " + err.Error()
    }
    
    // Check Databricks connection (simplified)
    databricksStatus := "healthy"
    if s.config.DatabricksHost == "" {
        databricksStatus = "unconfigured"
    }
    
    // Calculate uptime
    uptime := time.Since(s.config.StartTime).String()
    
    return &pb.HealthResponse{
        Status:  "healthy",
        Version: "1.0.0",
        Services: map[string]string{
            "database":   dbStatus,
            "databricks": databricksStatus,
            "catalog":    "healthy",
            "jobs":       fmt.Sprintf("%d active", s.jobRunner.ActiveJobs()),
        },
        Uptime: uptime,
    }, nil
}
```

## Key Features Implemented

### 1. **Configuration Management**
- Add, list, and remove BLADE data sources
- Store configuration in database
- Support for multiple data types

### 2. **Query Operations**
- Query BLADE data with filters
- Get specific items by ID
- Pagination support
- Order by support

### 3. **Data Transformation**
- Convert Databricks rows to proto format
- Handle different data types
- Preserve metadata and classification

### 4. **Error Handling**
- Proper gRPC status codes
- Detailed error messages
- Input validation

### 5. **Health Monitoring**
- Database connectivity check
- Service status reporting
- Uptime tracking

## Testing the Server

Create a simple test file to verify the server compiles:

```go
// server/blade_server/server_test.go
package blade_server

import (
    "testing"
    "blade-ingestion-service/server/utils"
    "gorm.io/gorm"
)

func TestNewServer(t *testing.T) {
    // Mock dependencies
    db := &gorm.DB{}
    config := &utils.Config{
        DatabricksHost: "http://localhost:8080",
        DatabricksToken: "test-token",
    }
    
    // Create server
    server := NewServer(db, config, nil)
    
    // Verify server is created
    if server == nil {
        t.Fatal("Expected server to be created")
    }
    
    if server.databricks == nil {
        t.Fatal("Expected databricks client to be initialized")
    }
}
```

## Next Steps

The server struct references several components we need to implement:
1. Ingestion endpoints (IngestBLADEItem, BulkIngestBLADE)
2. Job management endpoints (StartBLADESync, etc.)
3. Query job functionality

✅ Main server setup complete  
✅ Server struct implemented  
➡️ Continue to [03-ingestion-endpoints.md](03-ingestion-endpoints.md) to implement ingestion functionality