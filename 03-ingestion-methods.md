# Ingestion Methods Implementation

This guide explains how to implement BLADE data ingestion methods after setting up the basic server structure.

## Overview

The ingestion methods handle the core functionality of processing BLADE data items, validating input, and managing the data processing workflow. This implementation assumes catalog upload functionality has been removed (as per the design).

## Prerequisites

- Basic server structure implemented
- Database models created
- Generated protobuf code available
- Understanding of the BLADE data processing workflow

## Core Ingestion Method Implementation

### File: `server/blade_server/server.go` (Ingestion Methods Section)

Add these methods to your existing server struct:

```go
// ============= Data Ingestion Methods =============

// IngestBLADEItem processes a BLADE data item
// This is the main ingestion endpoint that validates input and processes data
func (s *Server) IngestBLADEItem(ctx context.Context, req *pb.BladeDatum) (*pb.CatalogUploadResponse, error) {
	start := time.Now()
	defer func() {
		s.logResponse("IngestBLADEItem", time.Since(start), nil)
	}()

	log.Printf("Processing BLADE item: %s from source: %s", req.Id, req.DataSource)

	// Input validation
	if err := s.validateBladeDatum(req); err != nil {
		return nil, err
	}

	// Find and validate data source
	dataSource, err := s.findDataSourceByName(req.DataSource)
	if err != nil {
		return nil, err
	}

	// Process the BLADE item
	result, err := s.processBLADEItem(req, dataSource)
	if err != nil {
		log.Printf("Error processing BLADE item %s: %v", req.Id, err)
		return nil, status.Error(codes.Internal, "failed to process BLADE item")
	}

	log.Printf("Successfully processed BLADE item %s from source %s", req.Id, req.DataSource)
	return result, nil
}

// processBLADEItem handles the core business logic for processing a BLADE item
func (s *Server) processBLADEItem(req *pb.BladeDatum, dataSource *datasource.DataSource) (*pb.CatalogUploadResponse, error) {
	// Extract and validate the item data
	itemData, err := s.extractBLADEItemData(req, dataSource)
	if err != nil {
		return nil, err
	}

	// Transform the data according to BLADE specifications
	transformedData, err := s.transformBLADEData(itemData)
	if err != nil {
		return nil, err
	}

	// Validate the transformed data
	if err := s.validateTransformedData(transformedData); err != nil {
		return nil, err
	}

	// Store processing metadata (optional)
	if err := s.storeProcessingMetadata(req, dataSource, transformedData); err != nil {
		log.Printf("Warning: Failed to store metadata for item %s: %v", req.Id, err)
		// Continue processing even if metadata storage fails
	}

	// TODO: catalog upload - implement catalog upload functionality
	// For now, return a success response indicating processing completed
	
	return &pb.CatalogUploadResponse{
		Status:    "success",
		Message:   "BLADE item processed successfully (catalog upload disabled)",
		ItemId:    req.Id,
		CatalogId: "", // Empty since catalog upload is disabled
	}, nil
}

// ============= Data Processing Helper Methods =============

// validateBladeDatum validates the incoming BLADE datum request
func (s *Server) validateBladeDatum(req *pb.BladeDatum) error {
	if req.DataSource == "" {
		return status.Error(codes.InvalidArgument, "dataSource is required")
	}

	if req.Id == "" {
		return status.Error(codes.InvalidArgument, "id is required")
	}

	// Validate ID format (adjust based on BLADE requirements)
	if len(req.Id) > 255 {
		return status.Error(codes.InvalidArgument, "id is too long (max 255 characters)")
	}

	// Validate data source name format
	if len(req.DataSource) > 100 {
		return status.Error(codes.InvalidArgument, "dataSource name is too long (max 100 characters)")
	}

	// Additional validation based on BLADE specifications
	if req.Json != nil {
		if err := s.validateJSONStructure(req.Json); err != nil {
			return status.Errorf(codes.InvalidArgument, "invalid JSON structure: %v", err)
		}
	}

	return nil
}

// extractBLADEItemData extracts and structures the raw BLADE item data
func (s *Server) extractBLADEItemData(req *pb.BladeDatum, dataSource *datasource.DataSource) (*BLADEItemData, error) {
	// Create a structured representation of the BLADE item
	itemData := &BLADEItemData{
		ID:         req.Id,
		DataSource: dataSource,
		GetFile:    req.GetFile,
		Timestamp:  time.Now().UTC(),
		Metadata:   make(map[string]interface{}),
	}

	// Extract JSON metadata if provided
	if req.Json != nil {
		itemData.Metadata = req.Json.AsMap()
	}

	// Add source-specific metadata
	itemData.Metadata["source_name"] = dataSource.Name
	itemData.Metadata["source_display_name"] = dataSource.DisplayName
	itemData.Metadata["integration_name"] = dataSource.IntegrationName
	itemData.Metadata["processing_timestamp"] = itemData.Timestamp.Format(time.RFC3339)

	// If file retrieval is requested, handle file data extraction
	if req.GetFile {
		fileData, err := s.extractFileData(req, dataSource)
		if err != nil {
			return nil, status.Errorf(codes.Internal, "failed to extract file data: %v", err)
		}
		itemData.FileData = fileData
	}

	return itemData, nil
}

// transformBLADEData applies BLADE-specific transformations to the data
func (s *Server) transformBLADEData(itemData *BLADEItemData) (*TransformedBLADEData, error) {
	transformed := &TransformedBLADEData{
		OriginalItem: itemData,
		ProcessedAt:  time.Now().UTC(),
		Transformations: make(map[string]interface{}),
	}

	// Apply data standardization
	if err := s.standardizeData(itemData, transformed); err != nil {
		return nil, err
	}

	// Apply data enrichment
	if err := s.enrichData(itemData, transformed); err != nil {
		return nil, err
	}

	// Apply data validation rules
	if err := s.applyValidationRules(itemData, transformed); err != nil {
		return nil, err
	}

	// Apply format conversions
	if err := s.applyFormatConversions(itemData, transformed); err != nil {
		return nil, err
	}

	return transformed, nil
}

// ============= Data Transformation Helpers =============

// standardizeData applies standardization rules to BLADE data
func (s *Server) standardizeData(itemData *BLADEItemData, transformed *TransformedBLADEData) error {
	// Standardize timestamps
	if timestamp, exists := itemData.Metadata["timestamp"]; exists {
		if timestampStr, ok := timestamp.(string); ok {
			if parsedTime, err := time.Parse(time.RFC3339, timestampStr); err == nil {
				transformed.Transformations["standardized_timestamp"] = parsedTime.UTC().Format(time.RFC3339)
			}
		}
	}

	// Standardize string fields (trim whitespace, normalize case)
	for key, value := range itemData.Metadata {
		if strValue, ok := value.(string); ok {
			transformed.Transformations[key+"_standardized"] = strings.TrimSpace(strValue)
		}
	}

	return nil
}

// enrichData adds additional context and computed fields
func (s *Server) enrichData(itemData *BLADEItemData, transformed *TransformedBLADEData) error {
	// Add processing context
	transformed.Transformations["processing_context"] = map[string]interface{}{
		"processor":    "BLADE Minimal Service",
		"version":      "1.0.0",
		"processed_at": transformed.ProcessedAt.Format(time.RFC3339),
		"source_uri":   itemData.DataSource.URI,
	}

	// Add computed fields based on data source type
	switch itemData.DataSource.IntegrationName {
	case "BLADE":
		transformed.Transformations["integration_type"] = "blade"
		transformed.Transformations["data_category"] = "operational"
	default:
		transformed.Transformations["integration_type"] = "generic"
		transformed.Transformations["data_category"] = "unknown"
	}

	return nil
}

// applyValidationRules applies business validation rules
func (s *Server) applyValidationRules(itemData *BLADEItemData, transformed *TransformedBLADEData) error {
	validationResults := make(map[string]interface{})

	// Validate required fields
	requiredFields := []string{"id", "data_source"}
	for _, field := range requiredFields {
		if field == "id" && itemData.ID == "" {
			validationResults[field+"_validation"] = "FAILED: field is required"
		} else if field == "data_source" && itemData.DataSource.Name == "" {
			validationResults[field+"_validation"] = "FAILED: field is required"
		} else {
			validationResults[field+"_validation"] = "PASSED"
		}
	}

	// Validate metadata completeness
	if len(itemData.Metadata) > 0 {
		validationResults["metadata_validation"] = "PASSED: metadata present"
	} else {
		validationResults["metadata_validation"] = "WARNING: no metadata provided"
	}

	transformed.Transformations["validation_results"] = validationResults

	return nil
}

// applyFormatConversions converts data to standard formats
func (s *Server) applyFormatConversions(itemData *BLADEItemData, transformed *TransformedBLADEData) error {
	conversions := make(map[string]interface{})

	// Convert numeric strings to actual numbers
	for key, value := range itemData.Metadata {
		if strValue, ok := value.(string); ok {
			// Try to convert to float
			if floatValue, err := strconv.ParseFloat(strValue, 64); err == nil {
				conversions[key+"_numeric"] = floatValue
			}
			// Try to convert to int
			if intValue, err := strconv.ParseInt(strValue, 10, 64); err == nil {
				conversions[key+"_integer"] = intValue
			}
		}
	}

	if len(conversions) > 0 {
		transformed.Transformations["format_conversions"] = conversions
	}

	return nil
}

// ============= File Handling Methods =============

// extractFileData handles file data extraction when getFile is true
func (s *Server) extractFileData(req *pb.BladeDatum, dataSource *datasource.DataSource) (*BLADEFileData, error) {
	if !req.GetFile {
		return nil, nil
	}

	// Mock file data extraction - in a real implementation, this would:
	// 1. Connect to the BLADE data source
	// 2. Retrieve the actual file data
	// 3. Validate file format and size
	// 4. Extract metadata from the file

	fileData := &BLADEFileData{
		FileName:    req.Id + ".json", // Mock filename
		FileSize:    1024,             // Mock file size
		ContentType: "application/json",
		Data:        []byte(`{"mock":"file","id":"` + req.Id + `"}`), // Mock content
		ExtractedAt: time.Now().UTC(),
	}

	log.Printf("Extracted file data for item %s: %s (%d bytes)", req.Id, fileData.FileName, fileData.FileSize)

	return fileData, nil
}

// ============= Metadata and Logging Methods =============

// storeProcessingMetadata stores metadata about the processing operation
func (s *Server) storeProcessingMetadata(req *pb.BladeDatum, dataSource *datasource.DataSource, transformedData *TransformedBLADEData) error {
	// Create processing record
	processingRecord := &ProcessingRecord{
		ItemID:       req.Id,
		DataSourceID: dataSource.ID,
		ProcessedAt:  transformedData.ProcessedAt,
		Status:       "completed",
		Metadata:     transformedData.Transformations,
	}

	// Store in database (if you have a processing_records table)
	// if err := s.DB.Create(processingRecord).Error; err != nil {
	//     return err
	// }

	log.Printf("Processing metadata stored for item %s", req.Id)
	return nil
}

// validateJSONStructure validates the structure of JSON metadata
func (s *Server) validateJSONStructure(jsonStruct *structpb.Struct) error {
	if jsonStruct == nil {
		return nil
	}

	// Validate that JSON doesn't exceed reasonable size limits
	jsonMap := jsonStruct.AsMap()
	if len(jsonMap) > 1000 { // Adjust limit as needed
		return errors.New("JSON structure too large (max 1000 fields)")
	}

	// Validate no deeply nested structures (prevent DoS)
	if err := s.validateJSONDepth(jsonMap, 0, 10); err != nil {
		return err
	}

	return nil
}

// validateJSONDepth prevents deeply nested JSON structures
func (s *Server) validateJSONDepth(data interface{}, currentDepth, maxDepth int) error {
	if currentDepth > maxDepth {
		return errors.New("JSON structure too deeply nested")
	}

	switch v := data.(type) {
	case map[string]interface{}:
		for _, value := range v {
			if err := s.validateJSONDepth(value, currentDepth+1, maxDepth); err != nil {
				return err
			}
		}
	case []interface{}:
		for _, item := range v {
			if err := s.validateJSONDepth(item, currentDepth+1, maxDepth); err != nil {
				return err
			}
		}
	}

	return nil
}

// validateTransformedData ensures the transformed data is valid
func (s *Server) validateTransformedData(data *TransformedBLADEData) error {
	if data == nil {
		return errors.New("transformed data is nil")
	}

	if data.OriginalItem == nil {
		return errors.New("original item data is missing")
	}

	if data.ProcessedAt.IsZero() {
		return errors.New("processing timestamp is missing")
	}

	return nil
}
```

## Supporting Data Structures

Add these data structures to your server file or create a separate models file:

```go
// ============= Data Structures =============

// BLADEItemData represents the structured form of a BLADE data item
type BLADEItemData struct {
	ID         string                 `json:"id"`
	DataSource *datasource.DataSource `json:"data_source"`
	GetFile    bool                   `json:"get_file"`
	Timestamp  time.Time              `json:"timestamp"`
	Metadata   map[string]interface{} `json:"metadata"`
	FileData   *BLADEFileData         `json:"file_data,omitempty"`
}

// TransformedBLADEData represents the result of data transformation
type TransformedBLADEData struct {
	OriginalItem    *BLADEItemData             `json:"original_item"`
	ProcessedAt     time.Time                  `json:"processed_at"`
	Transformations map[string]interface{}     `json:"transformations"`
}

// BLADEFileData represents file data extracted from BLADE sources
type BLADEFileData struct {
	FileName    string    `json:"file_name"`
	FileSize    int64     `json:"file_size"`
	ContentType string    `json:"content_type"`
	Data        []byte    `json:"data"`
	ExtractedAt time.Time `json:"extracted_at"`
}

// ProcessingRecord represents metadata about a processing operation
type ProcessingRecord struct {
	ID           uint                   `json:"id" gorm:"primaryKey"`
	ItemID       string                 `json:"item_id" gorm:"index"`
	DataSourceID uint                   `json:"data_source_id" gorm:"index"`
	ProcessedAt  time.Time              `json:"processed_at"`
	Status       string                 `json:"status"`
	Metadata     map[string]interface{} `json:"metadata" gorm:"type:jsonb"`
	CreatedAt    time.Time              `json:"created_at"`
	UpdatedAt    time.Time              `json:"updated_at"`
}
```

## Additional Required Imports

Add these imports to your server file:

```go
import (
	"errors"
	"strconv"
	"time"
	
	"google.golang.org/protobuf/types/known/structpb"
)
```

## Integration with Catalog Upload (Future)

When implementing catalog upload functionality, modify the `processBLADEItem` method:

```go
// Replace the TODO section with:
if catalogEnabled {
	catalogResponse, err := s.sendItemToCatalog(transformedData)
	if err != nil {
		return nil, status.Errorf(codes.Internal, "catalog upload failed: %v", err)
	}
	return catalogResponse, nil
}
```

## Error Handling Strategy

### 1. Input Validation Errors

```go
if req.DataSource == "" {
    return nil, status.Error(codes.InvalidArgument, "dataSource is required")
}
```

### 2. Business Logic Errors

```go
if err := s.processBLADEItem(req, dataSource); err != nil {
    log.Printf("Processing failed for item %s: %v", req.Id, err)
    return nil, status.Error(codes.Internal, "processing failed")
}
```

### 3. External Service Errors

```go
if err := s.extractFileData(req, dataSource); err != nil {
    return nil, status.Errorf(codes.Internal, "file extraction failed: %v", err)
}
```

## Testing the Implementation

### 1. Basic Ingestion Test

```bash
curl -X POST "http://localhost:9091/blade/ingest" \
  -H "Content-Type: application/json" \
  -d '{
    "dataSource": "test-source",
    "id": "test-item-123",
    "getFile": false,
    "json": {
      "category": "maintenance",
      "priority": "high"
    }
  }'
```

### 2. File Ingestion Test

```bash
curl -X POST "http://localhost:9091/blade/ingest" \
  -H "Content-Type: application/json" \
  -d '{
    "dataSource": "test-source",
    "id": "test-file-456",
    "getFile": true,
    "json": {
      "file_type": "maintenance_log"
    }
  }'
```

### 3. Error Case Tests

```bash
# Missing required field
curl -X POST "http://localhost:9091/blade/ingest" \
  -H "Content-Type: application/json" \
  -d '{
    "dataSource": "",
    "id": "test-item-789"
  }'

# Non-existent data source
curl -X POST "http://localhost:9091/blade/ingest" \
  -H "Content-Type: application/json" \
  -d '{
    "dataSource": "non-existent-source",
    "id": "test-item-789"
  }'
```

## Performance Considerations

### 1. Async Processing

For high-volume scenarios, consider making processing asynchronous:

```go
go func() {
    s.processBLADEItemAsync(req, dataSource)
}()
return &pb.CatalogUploadResponse{
    Status: "accepted",
    Message: "Item queued for processing",
    ItemId: req.Id,
}, nil
```

### 2. Batch Processing

For multiple items, implement batch processing:

```go
func (s *Server) IngestBLADEItemBatch(ctx context.Context, req *pb.BladeDatumBatch) (*pb.CatalogUploadBatchResponse, error) {
    // Process multiple items efficiently
}
```

### 3. Caching

Cache frequently accessed data sources:

```go
// Add to server struct
dataSourceCache map[string]*datasource.DataSource
cacheMutex      sync.RWMutex
```

## Monitoring and Observability

### 1. Metrics

Track processing metrics:

```go
// Processing time
start := time.Now()
defer func() {
    duration := time.Since(start)
    log.Printf("Processing took %v for item %s", duration, req.Id)
}()
```

### 2. Structured Logging

Use structured logging for better observability:

```go
log.Printf("BLADE_ITEM_PROCESSED id=%s source=%s duration=%v status=%s", 
    req.Id, req.DataSource, duration, "success")
```

## Next Steps

1. Implement comprehensive error handling
2. Add unit tests for all methods
3. Implement batch processing capabilities
4. Add metrics and monitoring
5. Integrate with catalog upload when ready
6. Add authentication/authorization
7. Implement rate limiting

This implementation provides a robust foundation for BLADE data ingestion with proper validation, transformation, and error handling.
