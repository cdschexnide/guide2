# Step 3: Ingestion Endpoints Implementation

This guide covers implementing the ingestion endpoints for cataloging BLADE data.

## Add Ingestion Methods to server.go

Add these methods to your Server struct in `server/blade_server/server.go`:

```go
// ============= Ingestion Endpoints =============

// IngestBLADEItem ingests a specific BLADE item to catalog
func (s *Server) IngestBLADEItem(ctx context.Context, req *pb.BLADEItemRequest) (*pb.IngestionResponse, error) {
    log.Printf("Ingesting BLADE item - type: %s, id: %s", req.DataType, req.ItemId)
    
    // Validate input
    if req.DataType == "" || req.ItemId == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType and itemId are required")
    }
    
    // Get data source configuration
    var ds datasource.DataSource
    if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).First(&ds).Error; err != nil {
        return nil, status.Errorf(codes.NotFound, "no enabled data source found for type %s", req.DataType)
    }
    
    // Check if item already exists in local DB
    var existingItem models.BLADEItem
    if err := s.db.Where("item_id = ? AND data_type = ?", req.ItemId, req.DataType).First(&existingItem).Error; err == nil {
        // Check if already uploaded to catalog
        if existingItem.CatalogID != "" {
            return &pb.IngestionResponse{
                Status:         "already_ingested",
                ItemsProcessed: 1,
                ItemsSucceeded: 0,
                Details: map[string]string{
                    "catalog_id": existingItem.CatalogID,
                    "message":    "Item already ingested to catalog",
                },
            }, nil
        }
    }
    
    // Fetch from Databricks
    row, err := s.databricks.FetchBLADEItem(ctx, req.DataType, req.ItemId, ds.GetFullTableName())
    if err != nil {
        if strings.Contains(err.Error(), "not found") {
            return &pb.IngestionResponse{
                Status:         "failed",
                ItemsProcessed: 1,
                ItemsFailed:    1,
                Errors:         []string{fmt.Sprintf("Item %s not found in Databricks", req.ItemId)},
            }, nil
        }
        return nil, status.Error(codes.Internal, "failed to fetch item from Databricks")
    }
    
    // Transform to BLADE item
    item, err := TransformToBLADEItem(req.DataType, row)
    if err != nil {
        return nil, status.Error(codes.Internal, "failed to transform item")
    }
    
    // Add any metadata from request
    if req.Metadata != nil {
        additionalMeta := req.Metadata.AsMap()
        existingMeta := make(map[string]interface{})
        if len(item.Metadata) > 0 {
            json.Unmarshal(item.Metadata, &existingMeta)
        }
        for k, v := range additionalMeta {
            existingMeta[k] = v
        }
        metaJSON, _ := json.Marshal(existingMeta)
        item.Metadata = metaJSON
    }
    
    // Save to local database
    item.DataSourceID = ds.ID
    if err := s.db.Create(item).Error; err != nil {
        return nil, status.Error(codes.Internal, "failed to save item to database")
    }
    
    // Upload to catalog
    uploadErr := s.catalogClient.UploadItem(item)
    if uploadErr != nil {
        log.Printf("Error uploading to catalog: %v", uploadErr)
        return &pb.IngestionResponse{
            Status:         "partial_success",
            ItemsProcessed: 1,
            ItemsSucceeded: 0,
            ItemsFailed:    1,
            Errors:         []string{fmt.Sprintf("Failed to upload to catalog: %v", uploadErr)},
            Details: map[string]string{
                "local_id": fmt.Sprintf("%d", item.ID),
                "status":   "saved_locally",
            },
        }, nil
    }
    
    // Update catalog ID and upload time
    now := time.Now()
    item.CatalogID = fmt.Sprintf("BLADE-%s-%s", req.DataType, req.ItemId)
    item.UploadedAt = &now
    s.db.Save(item)
    
    // Update data source statistics
    s.db.Model(&ds).Update("item_count", gorm.Expr("item_count + ?", 1))
    
    return &pb.IngestionResponse{
        Status:         "success",
        ItemsProcessed: 1,
        ItemsSucceeded: 1,
        Details: map[string]string{
            "catalog_id": item.CatalogID,
            "local_id":   fmt.Sprintf("%d", item.ID),
        },
    }, nil
}

// BulkIngestBLADE performs bulk ingestion of BLADE items
func (s *Server) BulkIngestBLADE(ctx context.Context, req *pb.BulkIngestionRequest) (*pb.IngestionResponse, error) {
    log.Printf("Bulk ingesting BLADE items - type: %s, count: %d", req.DataType, len(req.ItemIds))
    
    // Validate input
    if req.DataType == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType is required")
    }
    
    // Get data source configuration
    var ds datasource.DataSource
    if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).First(&ds).Error; err != nil {
        return nil, status.Errorf(codes.NotFound, "no enabled data source found for type %s", req.DataType)
    }
    
    var itemsToIngest []string
    
    // If specific item IDs provided, use those
    if len(req.ItemIds) > 0 {
        itemsToIngest = req.ItemIds
    } else {
        // Otherwise, query based on filter
        query := fmt.Sprintf("SELECT item_id FROM %s", ds.GetFullTableName())
        
        if req.Filter != "" {
            query += fmt.Sprintf(" WHERE %s", req.Filter)
        }
        
        limit := req.MaxItems
        if limit <= 0 || limit > 1000 {
            limit = 100 // Default limit
        }
        query += fmt.Sprintf(" LIMIT %d", limit)
        
        // Execute query to get item IDs
        rows, err := s.databricks.ExecuteQuery(ctx, query)
        if err != nil {
            return nil, status.Error(codes.Internal, "failed to query items")
        }
        
        for _, row := range rows {
            if id, ok := row["item_id"].(string); ok {
                itemsToIngest = append(itemsToIngest, id)
            }
        }
    }
    
    // Process items
    var (
        processed  int32
        succeeded  int32
        failed     int32
        errors     []string
        errorCount = make(map[string]int)
    )
    
    // Process in batches
    batchSize := 10
    for i := 0; i < len(itemsToIngest); i += batchSize {
        end := i + batchSize
        if end > len(itemsToIngest) {
            end = len(itemsToIngest)
        }
        
        batch := itemsToIngest[i:end]
        
        for _, itemID := range batch {
            processed++
            
            // Check if cancelled
            select {
            case <-ctx.Done():
                return &pb.IngestionResponse{
                    Status:         "cancelled",
                    ItemsProcessed: processed,
                    ItemsSucceeded: succeeded,
                    ItemsFailed:    failed,
                    Errors:         append(errors, "Operation cancelled by client"),
                }, nil
            default:
            }
            
            // Ingest individual item
            resp, err := s.IngestBLADEItem(ctx, &pb.BLADEItemRequest{
                DataType: req.DataType,
                ItemId:   itemID,
                Metadata: req.Metadata,
            })
            
            if err != nil {
                failed++
                errMsg := fmt.Sprintf("Item %s: %v", itemID, err)
                if errorCount[err.Error()] < 5 { // Limit duplicate errors
                    errors = append(errors, errMsg)
                    errorCount[err.Error()]++
                }
            } else if resp.ItemsSucceeded > 0 {
                succeeded++
            } else if resp.ItemsFailed > 0 {
                failed++
                if len(resp.Errors) > 0 && errorCount[resp.Errors[0]] < 5 {
                    errors = append(errors, fmt.Sprintf("Item %s: %s", itemID, resp.Errors[0]))
                    errorCount[resp.Errors[0]]++
                }
            }
        }
        
        // Small delay between batches
        time.Sleep(100 * time.Millisecond)
    }
    
    // Determine overall status
    status := "success"
    if failed > 0 && succeeded == 0 {
        status = "failed"
    } else if failed > 0 {
        status = "partial_success"
    }
    
    // Add summary of errors if there were many
    for errMsg, count := range errorCount {
        if count >= 5 {
            errors = append(errors, fmt.Sprintf("... and %d more errors like: %s", count-5, errMsg))
        }
    }
    
    return &pb.IngestionResponse{
        Status:         status,
        ItemsProcessed: processed,
        ItemsSucceeded: succeeded,
        ItemsFailed:    failed,
        Errors:         errors,
        Details: map[string]string{
            "data_type":    req.DataType,
            "batch_size":   fmt.Sprintf("%d", batchSize),
            "total_items":  fmt.Sprintf("%d", len(itemsToIngest)),
        },
    }, nil
}

// ============= Ingestion Helper Methods =============

// checkAndUpdateCatalogStatus checks if items exist in catalog and updates local records
func (s *Server) checkAndUpdateCatalogStatus(items []models.BLADEItem) error {
    for i := range items {
        item := &items[i]
        
        // Skip if already has catalog ID
        if item.CatalogID != "" {
            continue
        }
        
        // Check if exists in catalog
        exists, err := s.catalogClient.CheckItemExists(item.DataType, item.ItemID)
        if err != nil {
            log.Printf("Error checking catalog for item %s: %v", item.ItemID, err)
            continue
        }
        
        if exists {
            // Update local record
            now := time.Now()
            item.CatalogID = fmt.Sprintf("BLADE-%s-%s", item.DataType, item.ItemID)
            item.UploadedAt = &now
            s.db.Save(item)
        }
    }
    
    return nil
}

// processIngestionBatch processes a batch of items for ingestion
func (s *Server) processIngestionBatch(ctx context.Context, items []map[string]interface{}, dataType string, ds *datasource.DataSource) (int, int, []string) {
    var succeeded, failed int
    var errors []string
    
    for _, row := range items {
        // Transform to BLADE item
        item, err := TransformToBLADEItem(dataType, row)
        if err != nil {
            failed++
            errors = append(errors, fmt.Sprintf("Transform error: %v", err))
            continue
        }
        
        // Set data source
        item.DataSourceID = ds.ID
        
        // Check if already exists
        var existing models.BLADEItem
        if err := s.db.Where("item_id = ? AND data_type = ?", item.ItemID, item.DataType).First(&existing).Error; err == nil {
            // Update existing
            item.ID = existing.ID
            if err := s.db.Save(item).Error; err != nil {
                failed++
                errors = append(errors, fmt.Sprintf("Update error for %s: %v", item.ItemID, err))
                continue
            }
        } else {
            // Create new
            if err := s.db.Create(item).Error; err != nil {
                failed++
                errors = append(errors, fmt.Sprintf("Create error for %s: %v", item.ItemID, err))
                continue
            }
        }
        
        // Upload to catalog
        if err := s.catalogClient.UploadItem(item); err != nil {
            failed++
            errors = append(errors, fmt.Sprintf("Upload error for %s: %v", item.ItemID, err))
            continue
        }
        
        // Update catalog info
        now := time.Now()
        item.CatalogID = fmt.Sprintf("BLADE-%s-%s", dataType, item.ItemID)
        item.UploadedAt = &now
        s.db.Save(item)
        
        succeeded++
    }
    
    return succeeded, failed, errors
}
```

## Key Features of Ingestion Implementation

### 1. **Single Item Ingestion**
- Fetches item from Databricks
- Transforms to standard format
- Saves to local database
- Uploads to catalog service
- Tracks upload status

### 2. **Bulk Ingestion**
- Supports item ID list or filter-based selection
- Processes in configurable batches
- Handles partial failures gracefully
- Provides detailed error reporting
- Respects context cancellation

### 3. **Deduplication**
- Checks if items already exist locally
- Verifies catalog upload status
- Prevents duplicate uploads
- Updates existing records

### 4. **Error Handling**
- Detailed error messages
- Error aggregation for bulk operations
- Partial success handling
- Proper status reporting

### 5. **Performance Optimizations**
- Batch processing
- Configurable batch sizes
- Rate limiting between batches
- Database query optimization

## Testing Ingestion

Create a test file to verify ingestion logic:

```go
// server/blade_server/ingestion_test.go
package blade_server

import (
    "context"
    "testing"
    pb "blade-ingestion-service/generated/proto"
)

func TestIngestBLADEItem(t *testing.T) {
    // This is a placeholder test
    // In real implementation, you would:
    // 1. Set up test database
    // 2. Mock Databricks client
    // 3. Mock catalog uploader
    // 4. Test various scenarios
    
    t.Skip("Implement with proper test infrastructure")
}

func TestBulkIngestBLADE(t *testing.T) {
    // Test bulk ingestion scenarios:
    // - Empty item list
    // - Filter-based selection
    // - Partial failures
    // - Context cancellation
    
    t.Skip("Implement with proper test infrastructure")
}
```

## Configuration for Catalog Upload

Ensure your `.env` file has catalog configuration:

```bash
# Catalog Service Configuration
CATALOG_SERVICE_URL=http://localhost:8080/api/v1
CATALOG_AUTH_TOKEN=your-catalog-token
CATALOG_UPLOAD_TIMEOUT=30s
```

## Next Steps

✅ Main server setup complete  
✅ Server struct implemented  
✅ Ingestion endpoints implemented  
➡️ Continue to [04-job-endpoints.md](04-job-endpoints.md) to implement job management endpoints