# Step 3: Environment Configuration

This guide covers setting up the configuration management system following cataloger patterns.

## 1. Create Configuration Structure

Create `server/utils/config.go`:

```go
package utils

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

// Config holds all configuration for the BLADE Ingestion Service
type Config struct {
    // Server Configuration
    Host     string
    GRPCPort string
    RESTPort string
    
    // Database Configuration
    DBHost     string
    DBPort     string
    DBUser     string
    DBPassword string
    DBName     string
    DBSchema   string
    
    // Mock Databricks Configuration
    MockDatabricksURL    string
    MockDatabricksToken  string
    MockWarehouseID      string
    MockRequestTimeout   time.Duration
    
    // Catalog Configuration
    CatalogURL           string
    CatalogAuthToken     string
    CatalogTimeout       time.Duration
    CatalogBatchSize     int
    CatalogRetryAttempts int
    
    // Data Processing Configuration
    DefaultClassification string
    MaxRecordsPerQuery   int
    EnableDataValidation bool
    
    // BLADE-specific Configuration
    BLADEDataTypes []string
    DataTypeMapping map[string]string
    
    // Performance Configuration
    ConcurrentUploads  int
    RateLimitPerSecond int
    ProcessingTimeout  time.Duration
    
    // Logging
    LogLevel  string
    LogFormat string
    
    // Feature Flags
    UseSSL           bool
    EnableSwaggerUI  bool
    EnableMetrics    bool
}

// LoadConfig loads configuration from environment variables
func LoadConfig() (*Config, error) {
    config := &Config{
        // Server defaults
        Host:     getEnvOrDefault("HOST", "0.0.0.0"),
        GRPCPort: getEnvOrDefault("GRPC_PORT", "9090"),
        RESTPort: getEnvOrDefault("REST_PORT", "9091"),
        
        // Database
        DBHost:     getEnvOrDefault("PGHOST", "localhost"),
        DBPort:     getEnvOrDefault("PGPORT", "5432"),
        DBUser:     getEnvOrDefault("PG_USER", "blade_user"),
        DBPassword: os.Getenv("APP_DB_ADMIN_PASSWORD"),
        DBName:     getEnvOrDefault("PG_DATABASE", "blade_ingestion"),
        DBSchema:   getEnvOrDefault("DB_SCHEMA_NAME", "public"),
        
        // Mock Databricks
        MockDatabricksURL:   getEnvOrDefault("MOCK_DATABRICKS_URL", "http://localhost:8080"),
        MockDatabricksToken: os.Getenv("MOCK_DATABRICKS_TOKEN"),
        MockWarehouseID:     getEnvOrDefault("MOCK_DATABRICKS_WAREHOUSE_ID", "test-warehouse"),
        MockRequestTimeout:  getDurationOrDefault("MOCK_REQUEST_TIMEOUT", 30*time.Second),
        
        // Catalog
        CatalogURL:           getEnvOrDefault("CATALOG_URL", "http://localhost:8092"),
        CatalogAuthToken:     os.Getenv("CATALOG_AUTH_TOKEN"),
        CatalogTimeout:       getDurationOrDefault("CATALOG_TIMEOUT", 60*time.Second),
        CatalogBatchSize:     getIntOrDefault("CATALOG_BATCH_SIZE", 10),
        CatalogRetryAttempts: getIntOrDefault("CATALOG_RETRY_ATTEMPTS", 3),
        
        // Data processing
        DefaultClassification: getEnvOrDefault("DEFAULT_CLASSIFICATION", "UNCLASSIFIED"),
        MaxRecordsPerQuery:   getIntOrDefault("MAX_RECORDS_PER_QUERY", 1000),
        EnableDataValidation: getBoolOrDefault("ENABLE_DATA_VALIDATION", true),
        
        // BLADE data types
        BLADEDataTypes: []string{"maintenance", "sortie", "deployment", "logistics"},
        DataTypeMapping: map[string]string{
            "maintenance": "blade_maintenance_data",
            "sortie":      "blade_sortie_data",
            "deployment":  "blade_deployment_data",
            "logistics":   "blade_logistics_data",
        },
        
        // Performance
        ConcurrentUploads:  getIntOrDefault("CONCURRENT_UPLOADS", 5),
        RateLimitPerSecond: getIntOrDefault("RATE_LIMIT_PER_SECOND", 10),
        ProcessingTimeout:  getDurationOrDefault("PROCESSING_TIMEOUT", 5*time.Minute),
        
        // Logging
        LogLevel:  getEnvOrDefault("LOG_LEVEL", "debug"),
        LogFormat: getEnvOrDefault("LOG_FORMAT", "json"),
        
        // Features
        UseSSL:          getBoolOrDefault("USE_SSL", false),
        EnableSwaggerUI: getBoolOrDefault("ENABLE_SWAGGER_UI", true),
        EnableMetrics:   getBoolOrDefault("ENABLE_METRICS", false),
    }
    
    // Validate required configuration
    if err := validateConfig(config); err != nil {
        return nil, fmt.Errorf("configuration validation failed: %w", err)
    }
    
    return config, nil
}

// validateConfig ensures all required configuration is present
func validateConfig(config *Config) error {
    if config.DBPassword == "" {
        return fmt.Errorf("database password (APP_DB_ADMIN_PASSWORD) is required")
    }
    
    if config.MockDatabricksURL == "" {
        return fmt.Errorf("mock Databricks URL is required")
    }
    
    if config.CatalogURL == "" {
        return fmt.Errorf("catalog URL is required")
    }
    
    if config.MaxRecordsPerQuery <= 0 {
        return fmt.Errorf("max records per query must be positive")
    }
    
    if config.CatalogBatchSize <= 0 {
        return fmt.Errorf("catalog batch size must be positive")
    }
    
    return nil
}

// GetDatabricksQuery builds a SQL query for a BLADE data type
func (c *Config) GetDatabricksQuery(dataType string, filter string, limit int) string {
    tableName, exists := c.DataTypeMapping[dataType]
    if !exists {
        tableName = fmt.Sprintf("blade_%s_data", dataType)
    }
    
    query := fmt.Sprintf("SELECT * FROM %s.%s", c.DBSchema, tableName)
    
    if filter != "" {
        query += " WHERE " + filter
    }
    
    if limit > 0 {
        query += fmt.Sprintf(" LIMIT %d", limit)
    } else if c.MaxRecordsPerQuery > 0 {
        query += fmt.Sprintf(" LIMIT %d", c.MaxRecordsPerQuery)
    }
    
    return query
}

// GetCatalogDataSource returns the catalog-compatible data source name
func (c *Config) GetCatalogDataSource(dataType string) string {
    return fmt.Sprintf("BLADE Databricks: %s", dataType)
}

// IsBLADEDataType checks if the given type is valid
func (c *Config) IsBLADEDataType(dataType string) bool {
    for _, validType := range c.BLADEDataTypes {
        if validType == dataType {
            return true
        }
    }
    return false
}

// Helper functions

func getEnvOrDefault(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getIntOrDefault(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.Atoi(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

func getBoolOrDefault(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.ParseBool(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

func getDurationOrDefault(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if parsed, err := time.ParseDuration(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}
```

## 2. Create Query Configuration Types

Add to `server/utils/config.go`:

```go
// QueryConfig represents configuration for a specific query operation
type QueryConfig struct {
    DataType           string
    SQLQuery           string
    MaxResults         int
    IncludeMetadata    bool
    FilterCriteria     map[string]interface{}
    ClassificationFilter string
}

// NewQueryConfig creates a query configuration
func (c *Config) NewQueryConfig(dataType, sqlQuery string) *QueryConfig {
    return &QueryConfig{
        DataType:        dataType,
        SQLQuery:        sqlQuery,
        MaxResults:      c.MaxRecordsPerQuery,
        IncludeMetadata: true,
        FilterCriteria:  make(map[string]interface{}),
        ClassificationFilter: c.DefaultClassification,
    }
}

// CatalogUploadConfig represents configuration for catalog upload operations
type CatalogUploadConfig struct {
    BatchSize          int
    MaxRetries         int
    RetryDelay         time.Duration
    ValidateBeforeUpload bool
    SkipDuplicates     bool
    MetadataEnrichment bool
}

// NewCatalogUploadConfig creates upload configuration
func (c *Config) NewCatalogUploadConfig() *CatalogUploadConfig {
    return &CatalogUploadConfig{
        BatchSize:            c.CatalogBatchSize,
        MaxRetries:           c.CatalogRetryAttempts,
        RetryDelay:           2 * time.Second,
        ValidateBeforeUpload: c.EnableDataValidation,
        SkipDuplicates:       true,
        MetadataEnrichment:   true,
    }
}
```

## 3. Create Development Environment File

Create `.env` for local development:

```bash
# Copy from example
cp .env.example .env

# Edit .env and set development values
# Server Configuration
HOST=0.0.0.0
GRPC_PORT=9090
REST_PORT=9091

# Database Configuration
PGHOST=localhost
PGPORT=5432
PG_USER=blade_user
APP_DB_ADMIN_PASSWORD=blade_password
PG_DATABASE=blade_ingestion
DB_SCHEMA_NAME=public

# Mock Databricks Configuration
MOCK_DATABRICKS_URL=http://localhost:8080
MOCK_DATABRICKS_TOKEN=dev-mock-token
MOCK_DATABRICKS_WAREHOUSE_ID=dev-warehouse

# Catalog Service Configuration
CATALOG_URL=http://localhost:8092
CATALOG_AUTH_TOKEN=dev-catalog-token
CATALOG_TIMEOUT=60s
CATALOG_BATCH_SIZE=10
CATALOG_RETRY_ATTEMPTS=3

# Data Processing Configuration
DEFAULT_CLASSIFICATION=UNCLASSIFIED
MAX_RECORDS_PER_QUERY=100
ENABLE_DATA_VALIDATION=true

# Performance Configuration
CONCURRENT_UPLOADS=5
RATE_LIMIT_PER_SECOND=10
PROCESSING_TIMEOUT=5m

# Logging
LOG_LEVEL=debug
LOG_FORMAT=text

# Feature Flags
USE_SSL=false
ENABLE_SWAGGER_UI=true
ENABLE_METRICS=false
```

## 4. Create Config Test

Create `server/utils/config_test.go`:

```go
package utils

import (
    "os"
    "testing"
    "time"
    
    "github.com/stretchr/testify/assert"
)

func TestLoadConfig(t *testing.T) {
    // Set test environment variables
    os.Setenv("APP_DB_ADMIN_PASSWORD", "test-password")
    os.Setenv("MOCK_DATABRICKS_URL", "http://test-databricks:8080")
    os.Setenv("CATALOG_URL", "http://test-catalog:8092")
    
    // Load config
    config, err := LoadConfig()
    
    // Assert no error
    assert.NoError(t, err)
    assert.NotNil(t, config)
    
    // Test defaults
    assert.Equal(t, "9090", config.GRPCPort)
    assert.Equal(t, "9091", config.RESTPort)
    assert.Equal(t, "blade_ingestion", config.DBName)
    
    // Test environment overrides
    assert.Equal(t, "test-password", config.DBPassword)
    assert.Equal(t, "http://test-databricks:8080", config.MockDatabricksURL)
    
    // Test BLADE data types
    assert.Contains(t, config.BLADEDataTypes, "maintenance")
    assert.Contains(t, config.BLADEDataTypes, "sortie")
    assert.True(t, config.IsBLADEDataType("maintenance"))
    assert.False(t, config.IsBLADEDataType("invalid"))
}

func TestGetDatabricksQuery(t *testing.T) {
    config := &Config{
        DBSchema: "public",
        DataTypeMapping: map[string]string{
            "maintenance": "blade_maintenance_data",
        },
        MaxRecordsPerQuery: 1000,
    }
    
    // Test basic query
    query := config.GetDatabricksQuery("maintenance", "", 0)
    expected := "SELECT * FROM public.blade_maintenance_data LIMIT 1000"
    assert.Equal(t, expected, query)
    
    // Test with filter
    query = config.GetDatabricksQuery("maintenance", "priority = 'HIGH'", 10)
    expected = "SELECT * FROM public.blade_maintenance_data WHERE priority = 'HIGH' LIMIT 10"
    assert.Equal(t, expected, query)
}

func TestValidateConfig(t *testing.T) {
    // Test missing password
    config := &Config{
        MockDatabricksURL: "http://test",
        CatalogURL: "http://test",
        MaxRecordsPerQuery: 100,
        CatalogBatchSize: 10,
    }
    err := validateConfig(config)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "password")
    
    // Test valid config
    config.DBPassword = "password"
    err = validateConfig(config)
    assert.NoError(t, err)
}
```

## 5. Run Configuration Tests

```bash
# Run the config tests
go test -v ./server/utils/
```

## Best Practices Applied

### 1. **Environment-based Configuration**
- All config from environment variables
- Sensible defaults for development
- Clear separation of concerns

### 2. **Type Safety**
- Strongly typed configuration struct
- Helper methods for type conversion
- Validation at startup

### 3. **Flexibility**
- Easy to add new configuration
- Support for different data types
- Feature flags for gradual rollout

### 4. **Security**
- Sensitive values from environment only
- No hardcoded credentials
- Validation of required secrets

## Next Steps

✅ Project structure created  
✅ Dependencies installed  
✅ Configuration management implemented  
➡️ Continue to [04-file-migration.md](04-file-migration.md) to migrate existing files