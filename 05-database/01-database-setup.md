# Step 1: Database Setup and Configuration

This guide covers setting up PostgreSQL database connectivity and migrations.

## Create database/database.go

Create the main database initialization file:

```go
package database

import (
    "fmt"
    "log"
    "time"
    
    "blade-ingestion-service/server/utils"
    
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

// InitDB initializes the database connection
func InitDB(config *utils.Config) (*gorm.DB, error) {
    // Build DSN (Data Source Name)
    dsn := buildDSN(config)
    
    // Configure GORM logger
    gormLogger := logger.Default
    if config.LogLevel == "debug" {
        gormLogger = logger.Default.LogMode(logger.Info)
    } else if config.LogLevel == "error" {
        gormLogger = logger.Default.LogMode(logger.Error)
    } else {
        gormLogger = logger.Default.LogMode(logger.Warn)
    }
    
    // Open database connection
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: gormLogger,
        NowFunc: func() time.Time {
            return time.Now().UTC()
        },
        PrepareStmt: true, // Prepare statements for better performance
    })
    
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }
    
    // Get underlying SQL database
    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("failed to get database instance: %w", err)
    }
    
    // Configure connection pool
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    
    // Test connection
    if err := sqlDB.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }
    
    log.Println("Database connection established successfully")
    return db, nil
}

// buildDSN builds the PostgreSQL connection string
func buildDSN(config *utils.Config) string {
    // Basic DSN format
    dsn := fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s",
        config.DBHost,
        config.DBPort,
        config.DBUser,
        config.DBPassword,
        config.DBName,
    )
    
    // Add SSL mode
    if config.DBSSLMode != "" {
        dsn += fmt.Sprintf(" sslmode=%s", config.DBSSLMode)
    } else {
        dsn += " sslmode=disable"
    }
    
    // Add timezone
    dsn += " TimeZone=UTC"
    
    return dsn
}

// HealthCheck performs a database health check
func HealthCheck(db *gorm.DB) error {
    sqlDB, err := db.DB()
    if err != nil {
        return fmt.Errorf("failed to get database instance: %w", err)
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    if err := sqlDB.PingContext(ctx); err != nil {
        return fmt.Errorf("database ping failed: %w", err)
    }
    
    // Check if we can execute a simple query
    var result int
    if err := db.Raw("SELECT 1").Scan(&result).Error; err != nil {
        return fmt.Errorf("database query failed: %w", err)
    }
    
    return nil
}

// CloseDB closes the database connection gracefully
func CloseDB(db *gorm.DB) error {
    sqlDB, err := db.DB()
    if err != nil {
        return fmt.Errorf("failed to get database instance: %w", err)
    }
    
    if err := sqlDB.Close(); err != nil {
        return fmt.Errorf("failed to close database: %w", err)
    }
    
    log.Println("Database connection closed")
    return nil
}
```

## Create database/migrations.go

Create the migrations handler:

```go
package database

import (
    "fmt"
    "log"
    
    "blade-ingestion-service/database/datasource"
    "blade-ingestion-service/database/models"
    "blade-ingestion-service/server/job"
    
    "gorm.io/gorm"
)

// Migrate runs all database migrations
func Migrate(db *gorm.DB) error {
    log.Println("Running database migrations...")
    
    // List of models to migrate
    modelsToMigrate := []interface{}{
        // BLADE models
        &models.BLADEItem{},
        
        // Data source models
        &datasource.DataSource{},
        
        // Job models
        &job.JobRecord{},
    }
    
    // Run auto-migration for each model
    for _, model := range modelsToMigrate {
        if err := db.AutoMigrate(model); err != nil {
            return fmt.Errorf("failed to migrate %T: %w", model, err)
        }
        log.Printf("Migrated model: %T", model)
    }
    
    // Create indexes
    if err := createIndexes(db); err != nil {
        return fmt.Errorf("failed to create indexes: %w", err)
    }
    
    // Seed initial data if needed
    if err := seedData(db); err != nil {
        return fmt.Errorf("failed to seed data: %w", err)
    }
    
    log.Println("Database migrations completed successfully")
    return nil
}

// createIndexes creates custom indexes for better performance
func createIndexes(db *gorm.DB) error {
    // Composite indexes for BLADE items
    if err := db.Exec(`
        CREATE INDEX IF NOT EXISTS idx_blade_items_type_modified 
        ON blade_items(data_type, last_modified DESC)
    `).Error; err != nil {
        return fmt.Errorf("failed to create blade_items index: %w", err)
    }
    
    // Index for job queries
    if err := db.Exec(`
        CREATE INDEX IF NOT EXISTS idx_jobs_type_status_time 
        ON jobs(job_type, status, start_time DESC)
    `).Error; err != nil {
        return fmt.Errorf("failed to create jobs index: %w", err)
    }
    
    // Index for data source lookups
    if err := db.Exec(`
        CREATE INDEX IF NOT EXISTS idx_data_sources_enabled 
        ON data_sources(enabled, data_type)
    `).Error; err != nil {
        return fmt.Errorf("failed to create data_sources index: %w", err)
    }
    
    log.Println("Custom indexes created successfully")
    return nil
}

// seedData seeds initial data if needed
func seedData(db *gorm.DB) error {
    // Check if we need to seed data
    var count int64
    db.Model(&datasource.DataSource{}).Count(&count)
    
    if count > 0 {
        // Data already exists
        return nil
    }
    
    log.Println("Seeding initial data...")
    
    // Seed default data sources
    defaultSources := []datasource.DataSource{
        {
            TypeName:    "blade-maintenance-default",
            DisplayName: "Default BLADE Maintenance Data",
            DataType:    "maintenance",
            Enabled:     true,
            WarehouseID: "default-warehouse",
            CatalogName: "hive_metastore",
            SchemaName:  "default",
            TableName:   "blade_maintenance",
        },
        {
            TypeName:    "blade-sortie-default",
            DisplayName: "Default BLADE Sortie Data",
            DataType:    "sortie",
            Enabled:     false,
            WarehouseID: "default-warehouse",
            CatalogName: "hive_metastore",
            SchemaName:  "default",
            TableName:   "blade_sortie",
        },
    }
    
    for _, source := range defaultSources {
        // Set default parameters
        params := map[string]interface{}{
            "sync_interval": "1h",
            "batch_size":    100,
        }
        
        if err := source.SetParameters(params); err != nil {
            return fmt.Errorf("failed to set parameters for %s: %w", source.TypeName, err)
        }
        
        if err := db.Create(&source).Error; err != nil {
            return fmt.Errorf("failed to create data source %s: %w", source.TypeName, err)
        }
        
        log.Printf("Created default data source: %s", source.TypeName)
    }
    
    return nil
}

// ResetDatabase drops all tables and recreates them (use with caution!)
func ResetDatabase(db *gorm.DB) error {
    log.Println("WARNING: Resetting database - all data will be lost!")
    
    // Drop tables in reverse order of dependencies
    tables := []string{
        "blade_items",
        "jobs",
        "data_sources",
    }
    
    for _, table := range tables {
        if err := db.Exec(fmt.Sprintf("DROP TABLE IF EXISTS %s CASCADE", table)).Error; err != nil {
            log.Printf("Warning: failed to drop table %s: %v", table, err)
        }
    }
    
    // Run migrations to recreate tables
    return Migrate(db)
}
```

## Create Database Test Utilities

Create `database/test_utils.go`:

```go
package database

import (
    "fmt"
    "testing"
    
    "blade-ingestion-service/server/utils"
    
    "gorm.io/gorm"
)

// SetupTestDB creates a test database connection
func SetupTestDB(t *testing.T) (*gorm.DB, func()) {
    // Create test config
    config := &utils.Config{
        DBHost:     "localhost",
        DBPort:     "5432",
        DBUser:     "postgres",
        DBPassword: "postgres",
        DBName:     fmt.Sprintf("blade_test_%d", time.Now().Unix()),
        DBSSLMode:  "disable",
    }
    
    // Connect to postgres database to create test DB
    masterDSN := fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=postgres sslmode=disable",
        config.DBHost,
        config.DBPort,
        config.DBUser,
        config.DBPassword,
    )
    
    masterDB, err := gorm.Open(postgres.Open(masterDSN), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        t.Fatalf("Failed to connect to master database: %v", err)
    }
    
    // Create test database
    if err := masterDB.Exec(fmt.Sprintf("CREATE DATABASE %s", config.DBName)).Error; err != nil {
        t.Fatalf("Failed to create test database: %v", err)
    }
    
    // Close master connection
    masterSQL, _ := masterDB.DB()
    masterSQL.Close()
    
    // Connect to test database
    db, err := InitDB(config)
    if err != nil {
        t.Fatalf("Failed to initialize test database: %v", err)
    }
    
    // Run migrations
    if err := Migrate(db); err != nil {
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    // Return cleanup function
    cleanup := func() {
        // Close connection
        CloseDB(db)
        
        // Reconnect to master to drop test DB
        masterDB, _ := gorm.Open(postgres.Open(masterDSN), &gorm.Config{
            Logger: logger.Default.LogMode(logger.Silent),
        })
        masterDB.Exec(fmt.Sprintf("DROP DATABASE IF EXISTS %s", config.DBName))
        masterSQL, _ := masterDB.DB()
        masterSQL.Close()
    }
    
    return db, cleanup
}

// SeedTestData seeds test data into the database
func SeedTestData(db *gorm.DB) error {
    // Add test BLADE items
    testItems := []models.BLADEItem{
        {
            ItemID:                "TEST-001",
            DataType:              "maintenance",
            Data:                  []byte(`{"test": "data"}`),
            ClassificationMarking: "UNCLASSIFIED",
            LastModified:          time.Now(),
        },
        {
            ItemID:                "TEST-002",
            DataType:              "sortie",
            Data:                  []byte(`{"test": "data2"}`),
            ClassificationMarking: "CONFIDENTIAL",
            LastModified:          time.Now(),
        },
    }
    
    for _, item := range testItems {
        if err := db.Create(&item).Error; err != nil {
            return fmt.Errorf("failed to create test item: %w", err)
        }
    }
    
    return nil
}
```

## Create Database Connection Test

Create `database/database_test.go`:

```go
package database

import (
    "testing"
    
    "blade-ingestion-service/server/utils"
)

func TestDatabaseConnection(t *testing.T) {
    // Skip if no database available
    config := &utils.Config{
        DBHost:     "localhost",
        DBPort:     "5432",
        DBUser:     "postgres",
        DBPassword: "postgres",
        DBName:     "postgres",
        DBSSLMode:  "disable",
    }
    
    db, err := InitDB(config)
    if err != nil {
        t.Skip("Skipping database test - no database available:", err)
    }
    defer CloseDB(db)
    
    // Test health check
    if err := HealthCheck(db); err != nil {
        t.Errorf("Health check failed: %v", err)
    }
}

func TestMigrations(t *testing.T) {
    db, cleanup := SetupTestDB(t)
    defer cleanup()
    
    // Migrations already run in SetupTestDB
    // Test that tables exist
    
    var count int64
    
    // Check BLADE items table
    if err := db.Raw("SELECT COUNT(*) FROM blade_items").Count(&count).Error; err != nil {
        t.Errorf("blade_items table not created: %v", err)
    }
    
    // Check data sources table
    if err := db.Raw("SELECT COUNT(*) FROM data_sources").Count(&count).Error; err != nil {
        t.Errorf("data_sources table not created: %v", err)
    }
    
    // Check jobs table
    if err := db.Raw("SELECT COUNT(*) FROM jobs").Count(&count).Error; err != nil {
        t.Errorf("jobs table not created: %v", err)
    }
}
```

## PostgreSQL Setup Instructions

### Local Development

1. **Install PostgreSQL**:
```bash
# macOS
brew install postgresql
brew services start postgresql

# Ubuntu/Debian
sudo apt-get install postgresql postgresql-contrib
sudo systemctl start postgresql

# Create database and user
createdb blade_ingestion
createuser blade_user
```

2. **Configure PostgreSQL**:
```sql
-- Connect as postgres user
psql -U postgres

-- Create user and database
CREATE USER blade_user WITH PASSWORD 'your_password';
CREATE DATABASE blade_ingestion OWNER blade_user;
GRANT ALL PRIVILEGES ON DATABASE blade_ingestion TO blade_user;

-- Exit
\q
```

3. **Update .env file**:
```bash
# Database Configuration
PGHOST=localhost
PGPORT=5432
PG_DATABASE=blade_ingestion
APP_DB_USER=blade_user
APP_DB_ADMIN_PASSWORD=your_password
DB_SSL_MODE=disable
```

### Docker Setup

Create `docker-compose.yml` for local development:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: blade_user
      POSTGRES_PASSWORD: blade_password
      POSTGRES_DB: blade_ingestion
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U blade_user -d blade_ingestion"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

Run with:
```bash
docker-compose up -d
```

## Database Schema Overview

### Tables Created

1. **blade_items**
   - Stores BLADE data items
   - Indexed by type and modification time
   - Tracks catalog upload status

2. **data_sources**
   - Configuration for BLADE data sources
   - Connection parameters
   - Sync status tracking

3. **jobs**
   - Async job tracking
   - Progress and status
   - Error logging

## Next Steps

✅ Database initialization complete  
✅ Migration system implemented  
✅ Test utilities created  
✅ PostgreSQL setup documented  
➡️ Continue to [06-testing/01-testing-guide.md](../06-testing/01-testing-guide.md) for testing setup