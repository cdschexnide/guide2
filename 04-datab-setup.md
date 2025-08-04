# Database Setup and Implementation

This guide explains how to implement the database layer after generating protobuf code, including models, migrations, and database initialization.

## Overview

The database layer provides data persistence for BLADE service configuration and operational data. It uses GORM as the ORM with PostgreSQL as the primary database, following patterns from infinityai-cataloger.

## Prerequisites

- PostgreSQL database server available
- GORM and PostgreSQL driver dependencies
- Understanding of Go struct tags and database modeling

## Database Structure

### Directory Structure

```
database/
├── database.go           # Main database initialization and migration
├── datasource/          # Data source models and operations
│   └── data_source.go   # DataSource model definition
└── migrations.go        # Additional migration utilities (optional)
```

## Core Database Implementation

### File: `database/database.go`

```go
package database

import (
	"fmt"
	"log"
	"os"
	"strconv"
	"time"

	"minimal-blade-ingestion/database/datasource"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

// DBInfo holds database connection information
type DBInfo struct {
	Host     string
	Port     string
	User     string
	Password string
	DbName   string
	Schema   string
}

// InitializeDatabase initializes the database connection and runs migrations
func InitializeDatabase() (*gorm.DB, error) {
	// Get database configuration from environment
	dbInfo, err := getDBInfoFromEnv()
	if err != nil {
		return nil, fmt.Errorf("failed to get database configuration: %w", err)
	}

	// Initialize database connection
	db, err := Initialize(dbInfo)
	if err != nil {
		return nil, fmt.Errorf("failed to initialize database: %w", err)
	}

	// Configure database connection pool
	if err := configureConnectionPool(db); err != nil {
		return nil, fmt.Errorf("failed to configure connection pool: %w", err)
	}

	// Run migrations
	if err := Migrate(db); err != nil {
		return nil, fmt.Errorf("failed to run migrations: %w", err)
	}

	// Run data seeding if needed
	if err := seedInitialData(db); err != nil {
		log.Printf("Warning: Failed to seed initial data: %v", err)
		// Don't fail startup for seeding errors
	}

	return db, nil
}

// Initialize creates a new database connection
func Initialize(dbInfo *DBInfo) (*gorm.DB, error) {
	// Build PostgreSQL connection string
	dsn := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%s sslmode=disable search_path=%s",
		dbInfo.Host, dbInfo.User, dbInfo.Password, dbInfo.DbName, dbInfo.Port, dbInfo.Schema)

	// Configure GORM logger based on environment
	var gormLogger logger.Interface
	if getEnv("LOG_LEVEL", "info") == "debug" {
		gormLogger = logger.Default.LogMode(logger.Info)
	} else {
		gormLogger = logger.Default.LogMode(logger.Silent)
	}

	// Open database connection
	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: gormLogger,
		NowFunc: func() time.Time {
			return time.Now().UTC()
		},
	})

	if err != nil {
		return nil, fmt.Errorf("failed to connect to database: %w", err)
	}

	log.Printf("Successfully connected to database: %s", dbInfo.DbName)
	return db, nil
}

// Migrate runs database migrations for all models
func Migrate(db *gorm.DB) error {
	log.Println("Running database migrations...")
	
	// Define all models that need to be migrated
	models := []interface{}{
		&datasource.DataSource{},
		// Add additional models here as they are created:
		// &processing.ProcessingRecord{},
		// &job.JobExecution{},
	}

	// Run auto-migration for all models
	err := db.AutoMigrate(models...)
	if err != nil {
		return fmt.Errorf("failed to run migrations: %w", err)
	}

	// Create indexes for better performance
	if err := createIndexes(db); err != nil {
		log.Printf("Warning: Failed to create some indexes: %v", err)
		// Don't fail migration for index creation errors
	}

	log.Println("Database migrations completed successfully")
	return nil
}

// configureConnectionPool sets up database connection pooling
func configureConnectionPool(db *gorm.DB) error {
	sqlDB, err := db.DB()
	if err != nil {
		return err
	}

	// Configure connection pool settings
	maxOpenConns, _ := strconv.Atoi(getEnv("DB_MAX_OPEN_CONNS", "25"))
	maxIdleConns, _ := strconv.Atoi(getEnv("DB_MAX_IDLE_CONNS", "5"))
	maxLifetime, _ := strconv.Atoi(getEnv("DB_MAX_LIFETIME_MINUTES", "60"))

	sqlDB.SetMaxOpenConns(maxOpenConns)
	sqlDB.SetMaxIdleConns(maxIdleConns)
	sqlDB.SetConnMaxLifetime(time.Duration(maxLifetime) * time.Minute)

	log.Printf("Database connection pool configured: max_open=%d, max_idle=%d, max_lifetime=%dm",
		maxOpenConns, maxIdleConns, maxLifetime)

	return nil
}

// createIndexes creates additional database indexes for performance
func createIndexes(db *gorm.DB) error {
	// Create indexes on frequently queried fields
	indexes := []string{
		"CREATE INDEX IF NOT EXISTS idx_data_sources_integration ON data_sources(integration_name)",
		"CREATE INDEX IF NOT EXISTS idx_data_sources_active ON data_sources(type_name) WHERE type_name IS NOT NULL",
		// Add more indexes as needed
	}

	for _, indexSQL := range indexes {
		if err := db.Exec(indexSQL).Error; err != nil {
			log.Printf("Failed to create index: %s, error: %v", indexSQL, err)
		}
	}

	return nil
}

// seedInitialData seeds the database with initial data if needed
func seedInitialData(db *gorm.DB) error {
	// Check if we should seed data
	if getEnv("SEED_INITIAL_DATA", "false") != "true" {
		return nil
	}

	log.Println("Seeding initial data...")

	// Check if data already exists
	var count int64
	if err := db.Model(&datasource.DataSource{}).Count(&count).Error; err != nil {
		return err
	}

	if count > 0 {
		log.Println("Initial data already exists, skipping seeding")
		return nil
	}

	// Create sample data sources
	sampleSources := []*datasource.DataSource{
		{
			Name:            "sample-blade-source",
			DisplayName:     "Sample BLADE Data Source",
			TypeName:        "sample-blade-source",
			URI:             "/blade/sample-blade-source",
			Description:     "Sample BLADE data source for testing",
			GetFile:         false,
			IntegrationName: "BLADE",
		},
		{
			Name:            "test-maintenance-source",
			DisplayName:     "Test Maintenance Data Source",
			TypeName:        "test-maintenance-source",
			URI:             "/blade/test-maintenance-source",
			Description:     "Test data source for maintenance data",
			GetFile:         true,
			IntegrationName: "BLADE",
		},
	}

	// Insert sample data
	for _, source := range sampleSources {
		if err := db.Create(source).Error; err != nil {
			log.Printf("Failed to create sample data source %s: %v", source.Name, err)
		}
	}

	log.Printf("Seeded %d initial data sources", len(sampleSources))
	return nil
}

// getDBInfoFromEnv retrieves database configuration from environment variables
func getDBInfoFromEnv() (*DBInfo, error) {
	dbInfo := &DBInfo{
		Host:     getEnv("PGHOST", "localhost"),
		Port:     getEnv("PGPORT", "5432"),
		User:     getEnv("PGUSER", "blade_user"),
		Password: getEnv("PGPASSWORD", ""),
		DbName:   getEnv("PG_DATABASE", "blade_db"),
		Schema:   getEnv("DB_SCHEMA", "public"),
	}

	// Validate required fields
	if dbInfo.Password == "" {
		return nil, fmt.Errorf("PGPASSWORD environment variable is required")
	}

	return dbInfo, nil
}

// getEnv gets environment variable with fallback default
func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

// HealthCheck checks if the database connection is healthy
func HealthCheck(db *gorm.DB) error {
	sqlDB, err := db.DB()
	if err != nil {
		return fmt.Errorf("failed to get database instance: %w", err)
	}

	if err := sqlDB.Ping(); err != nil {
		return fmt.Errorf("database ping failed: %w", err)
	}

	return nil
}

// GetDatabaseStats returns database connection statistics
func GetDatabaseStats(db *gorm.DB) (map[string]interface{}, error) {
	sqlDB, err := db.DB()
	if err != nil {
		return nil, err
	}

	stats := sqlDB.Stats()
	
	return map[string]interface{}{
		"max_open_connections": stats.MaxOpenConnections,
		"open_connections":     stats.OpenConnections,
		"in_use":              stats.InUse,
		"idle":                stats.Idle,
		"wait_count":          stats.WaitCount,
		"wait_duration":       stats.WaitDuration.String(),
		"max_idle_closed":     stats.MaxIdleClosed,
		"max_lifetime_closed": stats.MaxLifetimeClosed,
	}, nil
}
```

## Data Source Model Implementation

### File: `database/datasource/data_source.go`

```go
package datasource

import (
	"encoding/json"
	"time"

	"gorm.io/datatypes"
	"gorm.io/gorm"
)

// DataSource represents a configured BLADE data source
// This follows the infinityai-cataloger pattern exactly
type DataSource struct {
	ID              uint           `json:"id" gorm:"primaryKey;autoIncrement"`
	Name            string         `json:"name" gorm:"size:100;not null;index"`
	DisplayName     string         `json:"displayName" gorm:"size:200"`
	URI             string         `json:"uri" gorm:"size:500"`
	Description     string         `json:"description" gorm:"type:text"`
	Parameters      datatypes.JSON `json:"parameters" gorm:"type:jsonb"`
	TypeName        string         `json:"typeName" gorm:"size:100;uniqueIndex;not null"`
	GetFile         bool           `json:"getFile" gorm:"default:false"`
	IntegrationName string         `json:"integrationName" gorm:"size:50;not null;index;default:'BLADE'"`
	Active          bool           `json:"active" gorm:"default:true;index"`
	CreatedAt       time.Time      `json:"createdAt"`
	UpdatedAt       time.Time      `json:"updatedAt"`
	DeletedAt       gorm.DeletedAt `json:"deletedAt,omitempty" gorm:"index"`
}

// TableName specifies the table name for GORM
func (DataSource) TableName() string {
	return "data_sources"
}

// BeforeCreate is a GORM hook that runs before creating a record
func (ds *DataSource) BeforeCreate(tx *gorm.DB) error {
	// Ensure TypeName is set from Name if empty
	if ds.TypeName == "" {
		ds.TypeName = ds.Name
	}

	// Set default integration name
	if ds.IntegrationName == "" {
		ds.IntegrationName = "BLADE"
	}

	// Set default URI if empty
	if ds.URI == "" {
		ds.URI = "/blade/" + ds.TypeName
	}

	return nil
}

// BeforeUpdate is a GORM hook that runs before updating a record
func (ds *DataSource) BeforeUpdate(tx *gorm.DB) error {
	// Prevent changing TypeName after creation
	if tx.Statement.Changed("TypeName") {
		return gorm.ErrInvalidField
	}

	return nil
}

// SetParameters sets the parameters field as JSON
func (ds *DataSource) SetParameters(params map[string]interface{}) error {
	if params == nil {
		ds.Parameters = nil
		return nil
	}

	jsonData, err := json.Marshal(params)
	if err != nil {
		return err
	}
	ds.Parameters = jsonData
	return nil
}

// GetParameters retrieves the parameters as a map
func (ds *DataSource) GetParameters() (map[string]interface{}, error) {
	var params map[string]interface{}
	if len(ds.Parameters) == 0 {
		return make(map[string]interface{}), nil
	}
	
	err := json.Unmarshal(ds.Parameters, &params)
	if err != nil {
		return nil, err
	}
	return params, nil
}

// Validate performs validation on the data source
func (ds *DataSource) Validate() error {
	if ds.Name == "" {
		return fmt.Errorf("name is required")
	}

	if ds.TypeName == "" {
		return fmt.Errorf("typeName is required")
	}

	if len(ds.Name) > 100 {
		return fmt.Errorf("name is too long (max 100 characters)")
	}

	if len(ds.DisplayName) > 200 {
		return fmt.Errorf("displayName is too long (max 200 characters)")
	}

	return nil
}

// IsActive returns whether the data source is active
func (ds *DataSource) IsActive() bool {
	return ds.Active && ds.DeletedAt.Time.IsZero()
}

// SoftDelete performs a soft delete on the data source
func (ds *DataSource) SoftDelete(tx *gorm.DB) error {
	ds.Active = false
	return tx.Save(ds).Error
}

// ============= Repository Pattern (Optional) =============

// DataSourceRepository provides data access methods for DataSource
type DataSourceRepository struct {
	db *gorm.DB
}

// NewDataSourceRepository creates a new repository instance
func NewDataSourceRepository(db *gorm.DB) *DataSourceRepository {
	return &DataSourceRepository{db: db}
}

// Create creates a new data source
func (r *DataSourceRepository) Create(ds *DataSource) error {
	return r.db.Create(ds).Error
}

// GetByTypeName retrieves a data source by type name
func (r *DataSourceRepository) GetByTypeName(typeName string) (*DataSource, error) {
	var ds DataSource
	err := r.db.Where("type_name = ? AND active = ?", typeName, true).First(&ds).Error
	if err != nil {
		return nil, err
	}
	return &ds, nil
}

// GetByID retrieves a data source by ID
func (r *DataSourceRepository) GetByID(id uint) (*DataSource, error) {
	var ds DataSource
	err := r.db.First(&ds, id).Error
	if err != nil {
		return nil, err
	}
	return &ds, nil
}

// List retrieves all active data sources
func (r *DataSourceRepository) List() ([]*DataSource, error) {
	var sources []*DataSource
	err := r.db.Where("active = ?", true).Find(&sources).Error
	return sources, err
}

// Update updates an existing data source
func (r *DataSourceRepository) Update(ds *DataSource) error {
	return r.db.Save(ds).Error
}

// Delete soft deletes a data source
func (r *DataSourceRepository) Delete(id uint) error {
	return r.db.Delete(&DataSource{}, id).Error
}

// Count returns the total number of active data sources
func (r *DataSourceRepository) Count() (int64, error) {
	var count int64
	err := r.db.Model(&DataSource{}).Where("active = ?", true).Count(&count).Error
	return count, err
}

// FindByIntegration returns data sources by integration name
func (r *DataSourceRepository) FindByIntegration(integrationName string) ([]*DataSource, error) {
	var sources []*DataSource
	err := r.db.Where("integration_name = ? AND active = ?", integrationName, true).Find(&sources).Error
	return sources, err
}
```

## Additional Migration Utilities

### File: `database/migrations.go`

```go
package database

import (
	"fmt"
	"log"

	"gorm.io/gorm"
)

// MigrationFunc represents a database migration function
type MigrationFunc func(*gorm.DB) error

// Migration represents a single database migration
type Migration struct {
	ID          string
	Description string
	Up          MigrationFunc
	Down        MigrationFunc
}

// MigrationManager manages database migrations
type MigrationManager struct {
	db         *gorm.DB
	migrations []Migration
}

// NewMigrationManager creates a new migration manager
func NewMigrationManager(db *gorm.DB) *MigrationManager {
	return &MigrationManager{
		db:         db,
		migrations: make([]Migration, 0),
	}
}

// AddMigration adds a migration to the manager
func (mm *MigrationManager) AddMigration(migration Migration) {
	mm.migrations = append(mm.migrations, migration)
}

// RunMigrations runs all pending migrations
func (mm *MigrationManager) RunMigrations() error {
	// Create migrations table if it doesn't exist
	if err := mm.createMigrationsTable(); err != nil {
		return err
	}

	for _, migration := range mm.migrations {
		if err := mm.runMigration(migration); err != nil {
			return fmt.Errorf("failed to run migration %s: %w", migration.ID, err)
		}
	}

	return nil
}

// createMigrationsTable creates the migrations tracking table
func (mm *MigrationManager) createMigrationsTable() error {
	return mm.db.Exec(`
		CREATE TABLE IF NOT EXISTS schema_migrations (
			id VARCHAR(255) PRIMARY KEY,
			description TEXT,
			applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
		)
	`).Error
}

// runMigration runs a single migration if it hasn't been applied
func (mm *MigrationManager) runMigration(migration Migration) error {
	// Check if migration has already been applied
	var count int64
	err := mm.db.Raw("SELECT COUNT(*) FROM schema_migrations WHERE id = ?", migration.ID).Scan(&count).Error
	if err != nil {
		return err
	}

	if count > 0 {
		log.Printf("Migration %s already applied, skipping", migration.ID)
		return nil
	}

	// Run the migration
	log.Printf("Running migration: %s - %s", migration.ID, migration.Description)
	if err := migration.Up(mm.db); err != nil {
		return err
	}

	// Record the migration as applied
	return mm.db.Exec(
		"INSERT INTO schema_migrations (id, description) VALUES (?, ?)",
		migration.ID, migration.Description,
	).Error
}

// Sample migrations for BLADE service
func GetBLADEMigrations() []Migration {
	return []Migration{
		{
			ID:          "001_create_data_sources_table",
			Description: "Create data sources table with indexes",
			Up: func(db *gorm.DB) error {
				// This migration is handled by GORM AutoMigrate
				// But could include custom SQL for complex changes
				return nil
			},
		},
		{
			ID:          "002_add_data_source_indexes",
			Description: "Add performance indexes to data sources table",
			Up: func(db *gorm.DB) error {
				queries := []string{
					"CREATE INDEX IF NOT EXISTS idx_data_sources_integration ON data_sources(integration_name)",
					"CREATE INDEX IF NOT EXISTS idx_data_sources_active ON data_sources(active)",
					"CREATE INDEX IF NOT EXISTS idx_data_sources_created_at ON data_sources(created_at)",
				}

				for _, query := range queries {
					if err := db.Exec(query).Error; err != nil {
						return err
					}
				}
				return nil
			},
		},
	}
}
```

## Environment Variables

The database implementation expects these environment variables:

```bash
# Database connection
PGHOST=localhost
PGPORT=5432
PGUSER=blade_user
PGPASSWORD=blade_password
PG_DATABASE=blade_db
DB_SCHEMA=public

# Connection pooling
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_MAX_LIFETIME_MINUTES=60

# Development settings
LOG_LEVEL=info
SEED_INITIAL_DATA=false
```

## Docker Compose Database Service

Add this to your `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-blade_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-blade_password}
      POSTGRES_DB: ${DB_NAME:-blade_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-blade_user} -d ${DB_NAME:-blade_db}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

## Database Initialization Script

### File: `database/init.sql`

```sql
-- Initialize database for BLADE service
-- This script runs when the database container first starts

-- Create the database if needed (handled by POSTGRES_DB env var)
-- CREATE DATABASE blade_db;

-- Create extensions if needed
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create custom types if needed
-- CREATE TYPE blade_status AS ENUM ('active', 'inactive', 'pending');

-- Set timezone
SET timezone = 'UTC';

-- Create initial schema if needed (GORM will handle most of this)
-- The main tables will be created by GORM AutoMigrate
```

## Required Dependencies

Add these to your `go.mod`:

```go
require (
    gorm.io/gorm v1.30.0
    gorm.io/driver/postgres v1.6.0
    gorm.io/datatypes v1.2.5
)
```

## Testing the Database Setup

### 1. Connection Test

```go
func TestDatabaseConnection(t *testing.T) {
    db, err := database.InitializeDatabase()
    assert.NoError(t, err)
    assert.NotNil(t, db)
    
    err = database.HealthCheck(db)
    assert.NoError(t, err)
}
```

### 2. Migration Test

```go
func TestMigrations(t *testing.T) {
    db, err := database.InitializeDatabase()
    assert.NoError(t, err)
    
    // Test that tables were created
    assert.True(t, db.Migrator().HasTable(&datasource.DataSource{}))
}
```

### 3. Model Operations Test

```go
func TestDataSourceOperations(t *testing.T) {
    db, _ := database.InitializeDatabase()
    
    // Test create
    ds := &datasource.DataSource{
        Name:            "test-source",
        DisplayName:     "Test Source",
        IntegrationName: "BLADE",
    }
    
    err := db.Create(ds).Error
    assert.NoError(t, err)
    assert.NotZero(t, ds.ID)
    
    // Test read
    var retrieved datasource.DataSource
    err = db.First(&retrieved, ds.ID).Error
    assert.NoError(t, err)
    assert.Equal(t, ds.Name, retrieved.Name)
}
```

## Performance Considerations

### 1. Indexing Strategy

```sql
-- Add indexes for frequently queried fields
CREATE INDEX idx_data_sources_type_name ON data_sources(type_name);
CREATE INDEX idx_data_sources_integration ON data_sources(integration_name);
CREATE INDEX idx_data_sources_active ON data_sources(active);
CREATE INDEX idx_data_sources_created_at ON data_sources(created_at);
```

### 2. Connection Pooling

Configure appropriate connection pool settings based on load:

```go
sqlDB.SetMaxOpenConns(25)    // Adjust based on expected concurrent load
sqlDB.SetMaxIdleConns(5)     // Keep some connections idle
sqlDB.SetConnMaxLifetime(60 * time.Minute) // Rotate connections
```

### 3. Query Optimization

Use proper GORM methods to avoid N+1 queries:

```go
// Preload related data
db.Preload("RelatedModel").Find(&dataSources)

// Use specific columns
db.Select("id", "name", "type_name").Find(&dataSources)
```

## Security Considerations

1. **Environment Variables**: Never commit database passwords to code
2. **SQL Injection**: GORM provides protection, but be careful with raw queries
3. **Connection Security**: Use SSL in production
4. **Access Control**: Limit database user permissions appropriately

## Next Steps

1. Implement additional models as needed (ProcessingRecord, JobExecution, etc.)
2. Add comprehensive error handling
3. Implement database monitoring and health checks
4. Add backup and recovery procedures
5. Configure database for production deployment
6. Implement database connection retry logic
7. Add database metrics and monitoring

This database implementation provides a solid foundation for the BLADE service with proper modeling, migrations, and operational features.
