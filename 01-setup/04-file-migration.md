# Step 4: Migrate Existing Files

This guide covers migrating your existing code into the new project structure.

## Files to Migrate

From your existing structure:
- `config/databricks_config.go` → Already handled in `server/utils/config.go`
- `models/blade_models.go` → `database/models/blade_items.go`
- `databricks_integration.go` → Split into multiple files

## 1. Migrate BLADE Models

Create `database/models/blade_items.go`:

```go
package models

import (
    "time"
    "gorm.io/gorm"
    "gorm.io/datatypes"
)

// BLADEItemType represents the type of BLADE data
type BLADEItemType string

const (
    MaintenanceData BLADEItemType = "maintenance"
    SortieData      BLADEItemType = "sortie"
    DeploymentData  BLADEItemType = "deployment"
    LogisticsData   BLADEItemType = "logistics"
)

// BLADEItem represents a generic BLADE data item in the database
type BLADEItem struct {
    gorm.Model
    ItemID                string         `gorm:"uniqueIndex;not null" json:"item_id"`
    DataType              string         `gorm:"index;not null" json:"data_type"`
    Data                  datatypes.JSON `json:"data"`
    ClassificationMarking string         `json:"classification_marking"`
    LastModified          time.Time      `json:"last_modified"`
    Metadata              datatypes.JSON `json:"metadata,omitempty"`
    
    // Tracking fields
    DataSourceID   uint      `gorm:"index" json:"data_source_id"`
    IngestionJobID string    `gorm:"index" json:"ingestion_job_id,omitempty"`
    CatalogID      string    `json:"catalog_id,omitempty"`
    UploadedAt     *time.Time `json:"uploaded_at,omitempty"`
}

// TableName specifies the table name for BLADE items
func (BLADEItem) TableName() string {
    return "blade_items"
}

// BLADEMaintenanceData represents aircraft maintenance data
type BLADEMaintenanceData struct {
    ItemID                 string    `json:"item_id"`
    AircraftTail          string    `json:"aircraft_tail"`
    AircraftType          string    `json:"aircraft_type"`
    MaintenanceType       string    `json:"maintenance_type"`
    MaintenanceCode       string    `json:"maintenance_code"`
    Description           string    `json:"description"`
    Priority              string    `json:"priority"`
    EstimatedCompletion   *time.Time `json:"estimated_completion,omitempty"`
    ActualCompletion      *time.Time `json:"actual_completion,omitempty"`
    TechnicianAssigned    string    `json:"technician_assigned"`
    BaseLocation          string    `json:"base_location"`
    WorkOrder             string    `json:"work_order"`
    NextScheduledDate     *time.Time `json:"next_scheduled_date,omitempty"`
}

// BLADESortieData represents flight mission data
type BLADESortieData struct {
    ItemID              string    `json:"item_id"`
    MissionID           string    `json:"mission_id"`
    AircraftTail        string    `json:"aircraft_tail"`
    AircraftType        string    `json:"aircraft_type"`
    PilotCallsign       string    `json:"pilot_callsign"`
    MissionType         string    `json:"mission_type"`
    DepartureBase       string    `json:"departure_base"`
    DestinationBase     string    `json:"destination_base"`
    ScheduledDeparture  time.Time `json:"scheduled_departure"`
    ActualDeparture     *time.Time `json:"actual_departure,omitempty"`
    ScheduledArrival    time.Time `json:"scheduled_arrival"`
    ActualArrival       *time.Time `json:"actual_arrival,omitempty"`
    FlightHours         *float64  `json:"flight_hours,omitempty"`
    MissionStatus       string    `json:"mission_status"`
}

// BLADEDeploymentData represents military deployment information
type BLADEDeploymentData struct {
    ItemID               string    `json:"item_id"`
    DeploymentID         string    `json:"deployment_id"`
    UnitDesignation      string    `json:"unit_designation"`
    UnitType             string    `json:"unit_type"`
    PersonnelCount       int       `json:"personnel_count"`
    CommandingOfficer    string    `json:"commanding_officer"`
    DeploymentLocation   string    `json:"deployment_location"`
    OriginBase           string    `json:"origin_base"`
    DeploymentStartDate  time.Time `json:"deployment_start_date"`
    DeploymentEndDate    *time.Time `json:"deployment_end_date,omitempty"`
    MissionObjective     string    `json:"mission_objective"`
    OperationalStatus    string    `json:"operational_status"`
}

// BLADELogisticsData represents supply chain and logistics data
type BLADELogisticsData struct {
    ItemID              string    `json:"item_id"`
    ShipmentID          string    `json:"shipment_id"`
    SupplyType          string    `json:"supply_type"`
    Description         string    `json:"description"`
    Quantity            int       `json:"quantity"`
    UnitOfMeasure       string    `json:"unit_of_measure"`
    Vendor              string    `json:"vendor"`
    OriginLocation      string    `json:"origin_location"`
    DestinationLocation string    `json:"destination_location"`
    ShippedDate         *time.Time `json:"shipped_date,omitempty"`
    EstimatedArrival    *time.Time `json:"estimated_arrival,omitempty"`
    Priority            string    `json:"priority"`
}

// GetBLADEItemType determines the BLADE item type from a string
func GetBLADEItemType(itemType string) BLADEItemType {
    switch itemType {
    case "maintenance", "engine_maintenance", "avionics_check":
        return MaintenanceData
    case "sortie", "training_mission", "combat_mission":
        return SortieData
    case "deployment", "unit_deployment":
        return DeploymentData
    case "logistics", "supply_shipment", "parts_delivery":
        return LogisticsData
    default:
        return MaintenanceData // Default
    }
}

// GetDefaultClassification returns default classification for item type
func GetDefaultClassification(itemType BLADEItemType) string {
    switch itemType {
    case MaintenanceData:
        return "UNCLASSIFIED"
    case SortieData:
        return "CONFIDENTIAL"
    case DeploymentData:
        return "SECRET"
    case LogisticsData:
        return "UNCLASSIFIED"
    default:
        return "UNCLASSIFIED"
    }
}

// ValidateClassificationMarking validates classification marking
func ValidateClassificationMarking(marking string) bool {
    validMarkings := map[string]bool{
        "U":             true,
        "UNCLASSIFIED":  true,
        "CUI":           true,
        "C":             true,
        "CONFIDENTIAL":  true,
        "S":             true,
        "SECRET":        true,
        "TS":            true,
        "TOP SECRET":    true,
    }
    return validMarkings[marking]
}
```

## 2. Create Data Source Model

Create `database/datasource/data_source.go` (following cataloger pattern):

```go
package datasource

import (
    "time"
    "gorm.io/gorm"
    "gorm.io/datatypes"
)

// DataSource represents a configured BLADE data source
type DataSource struct {
    gorm.Model
    TypeName         string         `gorm:"uniqueIndex;not null" json:"type_name"`
    DisplayName      string         `json:"display_name"`
    DataType         string         `gorm:"index" json:"data_type"` // maintenance, sortie, etc.
    Enabled          bool           `json:"enabled"`
    Parameters       datatypes.JSON `json:"parameters"`
    
    // Sync configuration
    SyncEnabled      bool           `json:"sync_enabled"`
    SyncSchedule     string         `json:"sync_schedule,omitempty"` // Cron expression
    LastSyncTime     *time.Time     `json:"last_sync_time,omitempty"`
    LastSyncStatus   string         `json:"last_sync_status,omitempty"`
    
    // Statistics
    ItemCount        int            `json:"item_count"`
    LastErrorMessage string         `json:"last_error_message,omitempty"`
    
    // Databricks specific
    WarehouseID      string         `json:"warehouse_id"`
    CatalogName      string         `json:"catalog_name"`
    SchemaName       string         `json:"schema_name"`
    TableName        string         `json:"table_name"`
}

// GetParameters unmarshals parameters JSON
func (ds *DataSource) GetParameters() (map[string]interface{}, error) {
    var params map[string]interface{}
    if err := ds.Parameters.Scan(&params); err != nil {
        return nil, err
    }
    return params, nil
}

// SetParameters marshals parameters to JSON
func (ds *DataSource) SetParameters(params map[string]interface{}) error {
    data, err := datatypes.NewJSONType(params).MarshalJSON()
    if err != nil {
        return err
    }
    ds.Parameters = data
    return nil
}

// GetFullTableName returns the fully qualified table name
func (ds *DataSource) GetFullTableName() string {
    if ds.CatalogName != "" && ds.SchemaName != "" {
        return ds.CatalogName + "." + ds.SchemaName + "." + ds.TableName
    }
    return ds.TableName
}
```

## 3. Split Databricks Integration

The existing `databricks_integration.go` should be split into:

### A. Create `server/blade_server/databricks.go`:

```go
package blade_server

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
    
    "blade-ingestion-service/database/models"
    pb "blade-ingestion-service/generated/proto"
    
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// DatabricksClient handles communication with mock Databricks
type DatabricksClient struct {
    baseURL     string
    token       string
    warehouseID string
    httpClient  *http.Client
}

// NewDatabricksClient creates a new Databricks client
func NewDatabricksClient(baseURL, token, warehouseID string) *DatabricksClient {
    return &DatabricksClient{
        baseURL:     baseURL,
        token:       token,
        warehouseID: warehouseID,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// ExecuteQuery executes a SQL query against Databricks
func (dc *DatabricksClient) ExecuteQuery(ctx context.Context, query string) ([]map[string]interface{}, error) {
    // Prepare request
    reqBody := map[string]interface{}{
        "warehouse_id": dc.warehouseID,
        "statement":    query,
        "wait_timeout": "30s",
    }
    
    jsonData, err := json.Marshal(reqBody)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal request: %w", err)
    }
    
    // Create HTTP request
    req, err := http.NewRequestWithContext(ctx, "POST", 
        fmt.Sprintf("%s/api/2.0/sql/statements", dc.baseURL), 
        bytes.NewBuffer(jsonData))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }
    
    // Set headers
    req.Header.Set("Authorization", "Bearer "+dc.token)
    req.Header.Set("Content-Type", "application/json")
    
    // Execute request
    resp, err := dc.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to execute query: %w", err)
    }
    defer resp.Body.Close()
    
    // Read response
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("databricks returned status %d: %s", resp.StatusCode, string(body))
    }
    
    // Parse response
    var result struct {
        Result struct {
            Data [][]interface{} `json:"data"`
            Schema struct {
                Columns []struct {
                    Name string `json:"name"`
                } `json:"columns"`
            } `json:"schema"`
        } `json:"result"`
    }
    
    if err := json.Unmarshal(body, &result); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    // Convert to map format
    var rows []map[string]interface{}
    columns := result.Result.Schema.Columns
    
    for _, row := range result.Result.Data {
        rowMap := make(map[string]interface{})
        for i, col := range columns {
            if i < len(row) {
                rowMap[col.Name] = row[i]
            }
        }
        rows = append(rows, rowMap)
    }
    
    return rows, nil
}

// FetchBLADEItem fetches a specific BLADE item
func (dc *DatabricksClient) FetchBLADEItem(ctx context.Context, dataType, itemID string, tableName string) (map[string]interface{}, error) {
    query := fmt.Sprintf("SELECT * FROM %s WHERE item_id = '%s' LIMIT 1", tableName, itemID)
    
    rows, err := dc.ExecuteQuery(ctx, query)
    if err != nil {
        return nil, err
    }
    
    if len(rows) == 0 {
        return nil, fmt.Errorf("item not found")
    }
    
    return rows[0], nil
}

// TransformToBLADEItem converts raw data to BLADE item format
func TransformToBLADEItem(dataType string, rawData map[string]interface{}) (*models.BLADEItem, error) {
    // Generate item ID if not present
    itemID, ok := rawData["item_id"].(string)
    if !ok || itemID == "" {
        itemID = fmt.Sprintf("%s-%d", dataType, time.Now().Unix())
    }
    
    // Marshal data to JSON
    dataJSON, err := json.Marshal(rawData)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal data: %w", err)
    }
    
    // Determine classification
    classification := models.GetDefaultClassification(models.BLADEItemType(dataType))
    if class, ok := rawData["classification"].(string); ok {
        classification = class
    }
    
    // Create BLADE item
    item := &models.BLADEItem{
        ItemID:                itemID,
        DataType:              dataType,
        Data:                  dataJSON,
        ClassificationMarking: classification,
        LastModified:          time.Now(),
    }
    
    // Add metadata
    metadata := map[string]interface{}{
        "source":      "databricks",
        "import_time": time.Now().Format(time.RFC3339),
    }
    
    metadataJSON, _ := json.Marshal(metadata)
    item.Metadata = metadataJSON
    
    return item, nil
}
```

### B. Create `server/blade_server/helpers.go`:

```go
package blade_server

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "mime/multipart"
    "net/http"
    
    "blade-ingestion-service/database/models"
)

// CatalogUploader handles uploading items to the catalog
type CatalogUploader struct {
    catalogURL string
    authToken  string
    httpClient *http.Client
}

// NewCatalogUploader creates a new catalog uploader
func NewCatalogUploader(catalogURL, authToken string) *CatalogUploader {
    return &CatalogUploader{
        catalogURL: catalogURL,
        authToken:  authToken,
        httpClient: &http.Client{},
    }
}

// UploadItem uploads a BLADE item to the catalog
func (cu *CatalogUploader) UploadItem(item *models.BLADEItem) error {
    // Create multipart writer
    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)
    
    // Add file part
    fileName := fmt.Sprintf("%s_%s.json", item.DataType, item.ItemID)
    part, err := writer.CreateFormFile("file", fileName)
    if err != nil {
        return fmt.Errorf("failed to create form file: %w", err)
    }
    
    // Write JSON data
    if _, err := part.Write(item.Data); err != nil {
        return fmt.Errorf("failed to write data: %w", err)
    }
    
    // Add metadata fields
    writer.WriteField("dataSource", fmt.Sprintf("BLADE:%s", item.DataType))
    writer.WriteField("classificationMarking", item.ClassificationMarking)
    writer.WriteField("itemId", item.ItemID)
    
    // Add metadata JSON
    if len(item.Metadata) > 0 {
        writer.WriteField("metadata", string(item.Metadata))
    }
    
    // Close writer
    if err := writer.Close(); err != nil {
        return fmt.Errorf("failed to close writer: %w", err)
    }
    
    // Create request
    req, err := http.NewRequest("POST", cu.catalogURL+"/catalog/item", body)
    if err != nil {
        return fmt.Errorf("failed to create request: %w", err)
    }
    
    // Set headers
    req.Header.Set("Authorization", "Bearer "+cu.authToken)
    req.Header.Set("Content-Type", writer.FormDataContentType())
    
    // Send request
    resp, err := cu.httpClient.Do(req)
    if err != nil {
        return fmt.Errorf("failed to send request: %w", err)
    }
    defer resp.Body.Close()
    
    // Check response
    if resp.StatusCode != http.StatusOK && resp.StatusCode != http.StatusCreated {
        bodyBytes, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("catalog returned status %d: %s", resp.StatusCode, string(bodyBytes))
    }
    
    return nil
}

// CheckItemExists checks if an item already exists in the catalog
func (cu *CatalogUploader) CheckItemExists(dataType, itemID string) (bool, error) {
    url := fmt.Sprintf("%s/catalog/exists?source=%s&id=%s", cu.catalogURL, dataType, itemID)
    
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        return false, err
    }
    
    req.Header.Set("Authorization", "Bearer "+cu.authToken)
    
    resp, err := cu.httpClient.Do(req)
    if err != nil {
        return false, err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode == http.StatusOK {
        return true, nil
    } else if resp.StatusCode == http.StatusNotFound {
        return false, nil
    }
    
    return false, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
}

// BuildMetadata builds metadata for catalog upload
func BuildMetadata(item *models.BLADEItem, source string) map[string]interface{} {
    metadata := map[string]interface{}{
        "dataType":       item.DataType,
        "itemId":         item.ItemID,
        "classification": item.ClassificationMarking,
        "source":         source,
        "importTime":     item.CreatedAt.Format("2006-01-02T15:04:05Z"),
    }
    
    // Add any existing metadata
    if len(item.Metadata) > 0 {
        var existingMeta map[string]interface{}
        if err := json.Unmarshal(item.Metadata, &existingMeta); err == nil {
            for k, v := range existingMeta {
                metadata[k] = v
            }
        }
    }
    
    return metadata
}
```

## 4. Update Package Names

Make sure all files have the correct package names:

- Files in `server/blade_server/` → `package blade_server`
- Files in `database/models/` → `package models`
- Files in `database/datasource/` → `package datasource`
- Files in `server/utils/` → `package utils`

## 5. Remove Old Files

After migration, remove the old files:
```bash
# Remove old directories (if they exist)
rm -rf config/
rm -rf models/
rm databricks_integration.go
```

## Summary of Migration

✅ **Config**: Moved to `server/utils/config.go` with enhancements  
✅ **Models**: Moved to `database/models/blade_items.go` with GORM tags  
✅ **Integration**: Split into:
  - `server/blade_server/databricks.go` - Databricks client
  - `server/blade_server/helpers.go` - Catalog upload helpers
  - `server/blade_server/server.go` - Server implementation (next step)

## Next Steps

✅ Project structure created  
✅ Dependencies installed  
✅ Configuration management implemented  
✅ Existing files migrated  
➡️ Continue to [02-proto/01-proto-setup.md](../02-proto/01-proto-setup.md) to set up protocol buffers