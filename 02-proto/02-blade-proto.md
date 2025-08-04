# Step 2: Create BLADE Proto Definition

This guide covers creating the main protocol buffer definition for the BLADE ingestion service.

## Create blade_ingestion.proto

Create `proto/blade_ingestion.proto` with the complete service definition:

```protobuf
syntax = "proto3";
package blade;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";
import "protoc-gen-openapiv2/options/annotations.proto";

option go_package = "blade-ingestion-service/generated/proto";

// OpenAPI documentation
option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
  info: {
    title: "BLADE Ingestion Service API";
    description: "Service for ingesting BLADE (Basic Logistics and Deployment Engine) data from Databricks mock server into a catalog system.";
    version: "1.0";
    contact: {
      name: "BLADE Support";
      email: "blade-support@example.com";
    };
  };
  base_path: "";
  schemes: HTTP;
  schemes: HTTPS;
  consumes: "application/json";
  produces: "application/json";
  responses: {
    key: "404";
    value: {
      description: "Not found";
      schema: {
        json_schema: {
          ref: "#/definitions/rpcStatus";
        };
      };
    };
  };
};

// ============= Service Definition =============

service BLADEIngestionService {
  // ============= Configuration Endpoints =============
  
  // Add a BLADE data source
  rpc AddBLADESource(DataSource) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      post: "/configure/blade/{name}"
      body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Configuration";
      summary: "Configure a BLADE data source";
      description: "Adds a new Databricks data source for BLADE data. The source configuration includes connection details and data type mappings.";
    };
  }
  
  // List all configured BLADE data sources
  rpc ListBLADESources(google.protobuf.Empty) returns (DataSourceList) {
    option (google.api.http) = {
      get: "/configure/blade/sources"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Configuration";
      summary: "List all configured BLADE data sources";
      description: "Returns a list of all configured BLADE data sources with their current status and configuration.";
    };
  }
  
  // Remove a BLADE data source
  rpc RemoveBLADESource(DataSourceRequest) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      delete: "/configure/blade/{name}"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Configuration";
      summary: "Remove a BLADE data source";
      description: "Removes a configured BLADE data source. This does not delete any already ingested data.";
    };
  }
  
  // ============= Query Endpoints =============
  
  // Query BLADE data by type
  rpc QueryBLADE(BLADEQuery) returns (BLADEQueryResponse) {
    option (google.api.http) = {
      get: "/blade/{dataType}"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Query";
      summary: "Query BLADE data by type";
      description: "Queries BLADE data from Databricks based on data type with optional filtering and pagination.";
    };
  }
  
  // Get a specific BLADE item
  rpc GetBLADEItem(BLADEItemRequest) returns (BLADEItem) {
    option (google.api.http) = {
      get: "/blade/{dataType}/{itemId}"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Query";
      summary: "Get specific BLADE item by ID";
      description: "Retrieves a specific BLADE item by its unique identifier and data type.";
    };
  }
  
  // ============= Ingestion Endpoints =============
  
  // Ingest a specific BLADE item to catalog
  rpc IngestBLADEItem(BLADEItemRequest) returns (IngestionResponse) {
    option (google.api.http) = {
      post: "/blade/{dataType}/{itemId}/ingest"
      body: "metadata"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Ingestion";
      summary: "Ingest specific BLADE item to catalog";
      description: "Fetches a specific BLADE item from Databricks and ingests it into the catalog system.";
    };
  }
  
  // Bulk ingest BLADE items
  rpc BulkIngestBLADE(BulkIngestionRequest) returns (IngestionResponse) {
    option (google.api.http) = {
      post: "/blade/bulk-ingest"
      body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Ingestion";
      summary: "Bulk ingest BLADE items";
      description: "Ingests multiple BLADE items based on filter criteria or specific item IDs.";
    };
  }
  
  // ============= Job Management Endpoints =============
  
  // Start a BLADE sync job
  rpc StartBLADESync(SyncJobRequest) returns (JobResponse) {
    option (google.api.http) = {
      post: "/jobs/sync/start"
      body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Jobs";
      summary: "Start BLADE data sync job";
      description: "Starts an asynchronous job to sync BLADE data from Databricks to the catalog.";
    };
  }
  
  // Stop a running sync job
  rpc StopBLADESync(google.protobuf.Empty) returns (JobResponse) {
    option (google.api.http) = {
      post: "/jobs/sync/stop"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Jobs";
      summary: "Stop current sync job";
      description: "Stops the currently running synchronization job.";
    };
  }
  
  // Get sync job status
  rpc GetSyncStatus(google.protobuf.Empty) returns (SyncStatusResponse) {
    option (google.api.http) = {
      get: "/jobs/sync/status"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Jobs";
      summary: "Get current sync job status";
      description: "Returns detailed status information about the current or last sync job.";
    };
  }
  
  // Start async BLADE query job
  rpc StartBLADEQueryJob(BLADEQueryJobRequest) returns (JobResponse) {
    option (google.api.http) = {
      post: "/jobs/blade/query/start"
      body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Jobs";
      summary: "Start async BLADE query job";
      description: "Starts an asynchronous job to execute a custom SQL query against Databricks and ingest the results.";
    };
  }
  
  // Get BLADE query job status
  rpc GetBLADEQueryJobStatus(JobRequest) returns (JobStatusResponse) {
    option (google.api.http) = {
      get: "/jobs/blade/query/status/{jobId}"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Jobs";
      summary: "Get BLADE query job status";
      description: "Returns the current status and progress of a BLADE query job.";
    };
  }
  
  // ============= Health Check =============
  
  // Health check endpoint
  rpc HealthCheck(google.protobuf.Empty) returns (HealthResponse) {
    option (google.api.http) = {
      get: "/health"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "System";
      summary: "Service health check";
      description: "Returns the health status of the service and its dependencies.";
    };
  }
}

// ============= Message Definitions =============

// Configuration messages

message DataSource {
  string name = 1 [(google.api.field_behavior) = REQUIRED,
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Unique identifier for the data source (e.g., 'blade-maintenance')"
      example: "\"blade-maintenance\""
    }];
  
  string displayName = 2 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Human-readable name for the data source"
      example: "\"BLADE Maintenance Data\""
    }];
  
  string dataType = 3 [(google.api.field_behavior) = REQUIRED,
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Type of BLADE data: maintenance, sortie, deployment, or logistics"
      example: "\"maintenance\""
    }];
  
  bool enabled = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Whether this data source is currently active"
      example: "true"
    }];
  
  google.protobuf.Struct config = 5 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Additional configuration parameters (warehouse_id, catalog, schema, table)"
    }];
}

message DataSourceRequest {
  string name = 1 [(google.api.field_behavior) = REQUIRED];
}

message DataSourceList {
  repeated DataSource dataSources = 1;
}

// Query messages

message BLADEQuery {
  string dataType = 1 [(google.api.field_behavior) = REQUIRED,
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Type of BLADE data to query"
    }];
  
  string filter = 2 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Optional SQL WHERE clause for filtering"
      example: "\"priority = 'HIGH'\""
    }];
  
  int32 limit = 3 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Maximum number of results to return"
      example: "100"
    }];
  
  int32 offset = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Number of results to skip for pagination"
      example: "0"
    }];
  
  string orderBy = 5 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Sort order (e.g., 'created_at DESC')"
    }];
}

message BLADEQueryResponse {
  repeated BLADEItem items = 1;
  int32 totalCount = 2;
  string nextPageToken = 3;
}

// Item messages

message BLADEItem {
  string itemId = 1;
  string dataType = 2;
  google.protobuf.Struct data = 3;
  string classificationMarking = 4;
  google.protobuf.Timestamp lastModified = 5;
  map<string, string> metadata = 6;
}

message BLADEItemRequest {
  string dataType = 1 [(google.api.field_behavior) = REQUIRED];
  string itemId = 2 [(google.api.field_behavior) = REQUIRED];
  google.protobuf.Struct metadata = 3;
}

// Ingestion messages

message BulkIngestionRequest {
  string dataType = 1 [(google.api.field_behavior) = REQUIRED];
  repeated string itemIds = 2;
  string filter = 3;
  int32 maxItems = 4;
  map<string, string> metadata = 5;
}

message IngestionResponse {
  string status = 1;
  int32 itemsProcessed = 2;
  int32 itemsSucceeded = 3;
  int32 itemsFailed = 4;
  repeated string errors = 5;
  map<string, string> details = 6;
}

// Job messages

message SyncJobRequest {
  enum SyncType {
    FULL = 0;
    INCREMENTAL = 1;
    DATA_TYPE = 2;
  }
  
  SyncType syncType = 1;
  string dataType = 2;
  string filter = 3;
  int64 maxItems = 4;
  google.protobuf.Struct options = 5;
}

message BLADEQueryJobRequest {
  string sqlQuery = 1 [(google.api.field_behavior) = REQUIRED,
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "SQL query to execute against Databricks"
      example: "\"SELECT * FROM blade_maintenance_data WHERE priority = 'HIGH'\""
    }];
  
  string dataType = 2 [(google.api.field_behavior) = REQUIRED,
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Type of BLADE data being queried"
    }];
  
  map<string, string> parameters = 3 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Additional parameters for the query job"
    }];
  
  google.protobuf.Struct catalogConfig = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Configuration for catalog upload (classification, metadata)"
    }];
}

message JobRequest {
  string jobId = 1 [(google.api.field_behavior) = REQUIRED];
}

message JobResponse {
  string jobId = 1;
  string status = 2;
  string message = 3;
  google.protobuf.Timestamp startTime = 4;
}

message JobStatusResponse {
  string jobId = 1;
  string status = 2;
  float progress = 3;
  string currentOperation = 4;
  int32 totalItems = 5;
  int32 processedItems = 6;
  int32 successCount = 7;
  int32 errorCount = 8;
  google.protobuf.Timestamp startTime = 9;
  google.protobuf.Timestamp estimatedCompletion = 10;
  repeated string recentErrors = 11;
}

message SyncStatusResponse {
  string jobId = 1;
  string status = 2;
  string currentOperation = 3;
  int32 totalItems = 4;
  int32 processedItems = 5;
  int32 successCount = 6;
  int32 errorCount = 7;
  google.protobuf.Timestamp startTime = 8;
  google.protobuf.Timestamp estimatedCompletion = 9;
  repeated string recentErrors = 10;
  map<string, int32> progressByType = 11;
}

// System messages

message HealthResponse {
  string status = 1;
  string version = 2;
  map<string, string> services = 3;
  string uptime = 4;
}
```

## Key Features of This Proto

### 1. **Complete Service Definition**
- Configuration management endpoints
- Query endpoints for browsing data
- Ingestion endpoints for cataloging
- Job management for async operations
- Health check for monitoring

### 2. **OpenAPI Annotations**
- Swagger documentation auto-generated
- Field descriptions and examples
- HTTP method mappings
- Tag organization

### 3. **Field Behavior**
- Required fields marked with `field_behavior`
- Optional fields with defaults
- Clear documentation

### 4. **Flexible Data Model**
- Uses `google.protobuf.Struct` for flexible JSON
- Map types for metadata
- Timestamp fields for time tracking

### 5. **Job Management**
- Async query job support
- Sync job management
- Progress tracking

## Validation

Run a quick syntax check:
```bash
protoc --proto_path=proto proto/blade_ingestion.proto --descriptor_set_out=/dev/null
```

If no errors appear, the proto syntax is valid.

## Next Steps

✅ Proto dependencies set up  
✅ Main proto definition created  
➡️ Continue to [03-generate-code.md](03-generate-code.md) to generate the Go code