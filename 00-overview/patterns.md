# Key Patterns from InfinityAI-Cataloger

This document outlines the essential patterns we're borrowing from the infinityai-cataloger service.

## 1. Dual API Pattern (gRPC + REST)

### Pattern
```go
// Single proto definition
service BLADEIngestionService {
  rpc QueryBLADE(BLADEQuery) returns (BLADEQueryResponse) {
    option (google.api.http) = {
      get: "/blade/{dataType}"
    };
  }
}

// Generates both:
// - gRPC server interface
// - REST gateway handlers
```

### Benefits
- Single source of truth (proto file)
- Automatic REST endpoint generation
- Consistent behavior across protocols
- Built-in Swagger documentation

## 2. Job System Pattern

### Structure
```go
// Job Interface (from cataloger)
type JobFunction func(jc JobContext) error

// Job Context provides control
type JobContext struct {
    Ctx                   context.Context
    ProcessControlSignals chan Signal
    UpdateStatus          func(JobStatus, string)
    DB                    *gorm.DB
}

// Usage
var BLADEQueryJob JobFunction = func(jc JobContext) error {
    // Check for cancellation
    if jc.ShouldStop() {
        return nil
    }
    // Process data...
    return nil
}
```

### Key Features
- Pause/resume capability
- Progress tracking
- Graceful cancellation
- Database persistence

## 3. Configuration Handler Pattern

### Data Source Configuration
```go
// Add endpoint pattern
func (s *Server) AddBLADESource(ctx context.Context, req *pb.DataSource) (*emptypb.Empty, error) {
    // 1. Validate configuration
    // 2. Test connection
    // 3. Store in database
    // 4. Return success/error
}

// List endpoint pattern  
func (s *Server) ListBLADESources(ctx context.Context, req *emptypb.Empty) (*pb.DataSourceList, error) {
    var sources []datasource.DataSource
    s.DB.Find(&sources)
    // Convert to proto format
    return &pb.DataSourceList{DataSources: sources}, nil
}
```

## 4. Data Retrieval and Ingestion Pattern

### Flow
```go
// 1. Retrieve from external source
data, err := queryDatabricks(query)

// 2. Check if already cataloged
exists := catalogItemExists(itemID)
if exists {
    return AlreadyExistsError
}

// 3. Transform data
itemData := ItemData{
    Data:       data,
    DataSource: "BLADE Databricks",
    Name:       itemID + ".json",
}

// 4. Upload to catalog
resp := sendItemToCatalog(itemData, metadata)

// 5. Update tracking
updateDataSourceStats(dataSource, resp)
```

## 5. Server Initialization Pattern

### Server Structure
```go
type Server struct {
    pb.UnimplementedBLADEIngestionServiceServer
    DB        *gorm.DB
    JobRunner job.JobController
    Config    *utils.Config
}

// Initialization
func NewServer(db *gorm.DB, config *utils.Config) *Server {
    return &Server{
        DB:        db,
        JobRunner: job.NewJobRunner(),
        Config:    config,
    }
}
```

## 6. Error Handling Pattern

### Consistent Error Responses
```go
// gRPC errors
if err != nil {
    return nil, status.Errorf(codes.Internal, "failed to query: %v", err)
}

// Job errors - don't fail entire job
if err := processItem(item); err != nil {
    log.Printf("Error processing %s: %v", item.ID, err)
    continue // Process next item
}
```

## 7. Database Model Pattern

### GORM Models with Metadata
```go
type DataSource struct {
    gorm.Model
    TypeName       string `gorm:"unique;not null"`
    DisplayName    string
    Enabled        bool
    Parameters     datatypes.JSON
    LastSyncTime   *time.Time
    ItemCount      int
}

// JSON parameters for flexibility
params := map[string]interface{}{
    "warehouse_id": "test-warehouse",
    "catalog": "main",
    "schema": "blade",
}
```

## 8. Multipart Upload Pattern

### Catalog Upload
```go
func (s *Server) sendItemToCatalog(item ItemData, metadata map[string]interface{}) error {
    // Create multipart writer
    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)
    
    // Add file part
    part, _ := writer.CreateFormFile("file", item.Name)
    part.Write(item.Data)
    
    // Add metadata
    writer.WriteField("dataSource", item.DataSource)
    writer.WriteField("classificationMarking", metadata["classification"])
    
    // Send request
    req, _ := http.NewRequest("POST", catalogURL, body)
    req.Header.Set("Content-Type", writer.FormDataContentType())
    
    return client.Do(req)
}
```

## 9. Progress Tracking Pattern

### Real-time Updates
```go
// In job execution
for i, item := range items {
    // Process item
    processItem(item)
    
    // Update progress periodically
    if i % 10 == 0 {
        progress := float64(i) / float64(len(items)) * 100
        updateJobProgress(jobID, progress)
    }
}
```

## 10. Configuration Management Pattern

### Environment-based Config
```go
type Config struct {
    // Server
    GRPCPort string `env:"GRPC_PORT" default:"9090"`
    RESTPort string `env:"REST_PORT" default:"9091"`
    
    // Database
    DBHost string `env:"PGHOST" required:"true"`
    DBName string `env:"PG_DATABASE" default:"blade_ingestion"`
    
    // External Services
    DatabricksURL   string `env:"DATABRICKS_URL"`
    CatalogURL      string `env:"CATALOG_URL"`
}

// Load from environment
func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := envconfig.Process("", cfg); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

## Pattern Benefits

### 1. **Consistency**
- Same patterns across all endpoints
- Predictable behavior
- Easy to maintain

### 2. **Scalability**
- Job system handles large volumes
- Worker pool for concurrency
- Database tracking for reliability

### 3. **Flexibility**
- JSON parameters for configuration
- Multiple API access methods
- Extensible job types

### 4. **Observability**
- Progress tracking
- Error logging
- Status endpoints

## Anti-Patterns to Avoid

### 1. **Blocking Operations**
```go
// BAD: Blocks server
func (s *Server) IngestAll(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    for _, item := range millionItems {
        processItem(item) // Takes hours
    }
}

// GOOD: Use job system
func (s *Server) StartIngestion(ctx context.Context, req *pb.Request) (*pb.JobResponse, error) {
    job := NewIngestionJob(req)
    s.JobRunner.Submit(job)
    return &pb.JobResponse{JobId: job.ID()}, nil
}
```

### 2. **Tight Coupling**
```go
// BAD: Direct database access in handlers
func (s *Server) GetItem(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    db.Raw("SELECT * FROM items WHERE id = ?", req.Id).Scan(&item)
}

// GOOD: Use repository pattern
func (s *Server) GetItem(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    item, err := s.ItemRepo.FindByID(req.Id)
}
```

## Next Steps

Continue to [data-flow.md](data-flow.md) to understand how data moves through the system →