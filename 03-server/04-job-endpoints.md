# Step 4: Job Management Endpoints

This guide covers implementing the job management endpoints for async operations.

## Add Job Management Methods to server.go

Add these methods to your Server struct in `server/blade_server/server.go`:

```go
// ============= Job Management Endpoints =============

// StartBLADESync starts a BLADE sync job
func (s *Server) StartBLADESync(ctx context.Context, req *pb.SyncJobRequest) (*pb.JobResponse, error) {
    log.Printf("Starting BLADE sync job - type: %s, sync type: %s", req.DataType, req.SyncType)
    
    // Check if sync job already running
    if s.jobRunner.HasActiveJobOfType("blade_sync") {
        return nil, status.Error(codes.AlreadyExists, "sync job already running")
    }
    
    // Create job configuration
    jobConfig := map[string]interface{}{
        "sync_type": req.SyncType.String(),
        "data_type": req.DataType,
        "filter":    req.Filter,
        "max_items": req.MaxItems,
    }
    
    if req.Options != nil {
        for k, v := range req.Options.AsMap() {
            jobConfig[k] = v
        }
    }
    
    // Create sync job
    jobID := fmt.Sprintf("sync-%s-%d", req.DataType, time.Now().Unix())
    
    // Define the sync job function
    jobFunc := func(jc job.JobContext) error {
        return s.executeSyncJob(jc, req)
    }
    
    // Schedule the job
    if err := s.jobRunner.ScheduleJob(jobID, "blade_sync", jobFunc, jobConfig); err != nil {
        return nil, status.Error(codes.Internal, "failed to schedule sync job")
    }
    
    return &pb.JobResponse{
        JobId:     jobID,
        Status:    "started",
        Message:   fmt.Sprintf("Sync job started for %s data", req.DataType),
        StartTime: timestamppb.Now(),
    }, nil
}

// StopBLADESync stops the current sync job
func (s *Server) StopBLADESync(ctx context.Context, req *emptypb.Empty) (*pb.JobResponse, error) {
    log.Println("Stopping BLADE sync job")
    
    // Find active sync job
    activeJobs := s.jobRunner.GetActiveJobs()
    var syncJobID string
    
    for _, j := range activeJobs {
        if j.Type == "blade_sync" {
            syncJobID = j.ID
            break
        }
    }
    
    if syncJobID == "" {
        return nil, status.Error(codes.NotFound, "no active sync job found")
    }
    
    // Stop the job
    if err := s.jobRunner.CancelJob(syncJobID); err != nil {
        return nil, status.Error(codes.Internal, "failed to stop sync job")
    }
    
    return &pb.JobResponse{
        JobId:   syncJobID,
        Status:  "stopped",
        Message: "Sync job stopped",
    }, nil
}

// GetSyncStatus gets the current sync job status
func (s *Server) GetSyncStatus(ctx context.Context, req *emptypb.Empty) (*pb.SyncStatusResponse, error) {
    // Find sync job (active or most recent)
    jobs := s.jobRunner.GetJobsByType("blade_sync")
    if len(jobs) == 0 {
        return &pb.SyncStatusResponse{
            Status: "no_sync_job",
        }, nil
    }
    
    // Get most recent job
    var mostRecent *job.JobInfo
    for _, j := range jobs {
        if mostRecent == nil || j.StartTime.After(mostRecent.StartTime) {
            mostRecent = &j
        }
    }
    
    if mostRecent == nil {
        return &pb.SyncStatusResponse{
            Status: "no_sync_job",
        }, nil
    }
    
    // Get job progress
    progress := s.jobRunner.GetJobProgress(mostRecent.ID)
    
    // Calculate progress by type
    progressByType := make(map[string]int32)
    if typeProgress, ok := progress["types"].(map[string]interface{}); ok {
        for k, v := range typeProgress {
            if count, ok := v.(int); ok {
                progressByType[k] = int32(count)
            }
        }
    }
    
    return &pb.SyncStatusResponse{
        JobId:             mostRecent.ID,
        Status:            mostRecent.Status,
        CurrentOperation:  getStringFromProgress(progress, "current_operation"),
        TotalItems:        getInt32FromProgress(progress, "total_items"),
        ProcessedItems:    getInt32FromProgress(progress, "processed_items"),
        SuccessCount:      getInt32FromProgress(progress, "success_count"),
        ErrorCount:        getInt32FromProgress(progress, "error_count"),
        StartTime:         timestamppb.New(mostRecent.StartTime),
        EstimatedCompletion: estimateCompletion(mostRecent.StartTime, progress),
        RecentErrors:      getErrorsFromProgress(progress),
        ProgressByType:    progressByType,
    }, nil
}

// StartBLADEQueryJob starts an async BLADE query job
func (s *Server) StartBLADEQueryJob(ctx context.Context, req *pb.BLADEQueryJobRequest) (*pb.JobResponse, error) {
    log.Printf("Starting BLADE query job - type: %s", req.DataType)
    
    // Validate input
    if req.SqlQuery == "" {
        return nil, status.Error(codes.InvalidArgument, "sqlQuery is required")
    }
    
    if req.DataType == "" {
        return nil, status.Error(codes.InvalidArgument, "dataType is required")
    }
    
    // Create job configuration
    jobConfig := map[string]interface{}{
        "sql_query":  req.SqlQuery,
        "data_type":  req.DataType,
        "parameters": req.Parameters,
    }
    
    if req.CatalogConfig != nil {
        jobConfig["catalog_config"] = req.CatalogConfig.AsMap()
    }
    
    // Create query job
    jobID := fmt.Sprintf("query-%s-%d", req.DataType, time.Now().Unix())
    
    // Define the query job function
    jobFunc := func(jc job.JobContext) error {
        return s.executeQueryJob(jc, req)
    }
    
    // Schedule the job
    if err := s.jobRunner.ScheduleJob(jobID, "blade_query", jobFunc, jobConfig); err != nil {
        return nil, status.Error(codes.Internal, "failed to schedule query job")
    }
    
    return &pb.JobResponse{
        JobId:     jobID,
        Status:    "started",
        Message:   "Query job started",
        StartTime: timestamppb.Now(),
    }, nil
}

// GetBLADEQueryJobStatus gets the status of a BLADE query job
func (s *Server) GetBLADEQueryJobStatus(ctx context.Context, req *pb.JobRequest) (*pb.JobStatusResponse, error) {
    log.Printf("Getting BLADE query job status - id: %s", req.JobId)
    
    if req.JobId == "" {
        return nil, status.Error(codes.InvalidArgument, "jobId is required")
    }
    
    // Get job info
    jobInfo, err := s.jobRunner.GetJobInfo(req.JobId)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "job %s not found", req.JobId)
    }
    
    // Get job progress
    progress := s.jobRunner.GetJobProgress(req.JobId)
    
    // Calculate estimated completion
    var estimatedCompletion *timestamppb.Timestamp
    if jobInfo.Status == "running" {
        estimatedCompletion = estimateCompletion(jobInfo.StartTime, progress)
    }
    
    return &pb.JobStatusResponse{
        JobId:               jobInfo.ID,
        Status:              jobInfo.Status,
        Progress:            getFloatFromProgress(progress, "percent_complete"),
        CurrentOperation:    getStringFromProgress(progress, "current_operation"),
        TotalItems:          getInt32FromProgress(progress, "total_items"),
        ProcessedItems:      getInt32FromProgress(progress, "processed_items"),
        SuccessCount:        getInt32FromProgress(progress, "success_count"),
        ErrorCount:          getInt32FromProgress(progress, "error_count"),
        StartTime:           timestamppb.New(jobInfo.StartTime),
        EstimatedCompletion: estimatedCompletion,
        RecentErrors:        getErrorsFromProgress(progress),
    }, nil
}

// ============= Job Execution Methods =============

// executeSyncJob executes a sync job
func (s *Server) executeSyncJob(jc job.JobContext, req *pb.SyncJobRequest) error {
    log.Printf("Executing sync job: %s", jc.JobID())
    
    // Update job progress
    jc.UpdateProgress(map[string]interface{}{
        "current_operation": "Initializing sync",
        "percent_complete":  0,
    })
    
    // Determine what to sync
    var dataSources []datasource.DataSource
    
    if req.DataType != "" {
        // Sync specific data type
        if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).Find(&dataSources).Error; err != nil {
            return fmt.Errorf("failed to get data sources: %w", err)
        }
    } else {
        // Sync all enabled data sources
        if err := s.db.Where("enabled = ?", true).Find(&dataSources).Error; err != nil {
            return fmt.Errorf("failed to get data sources: %w", err)
        }
    }
    
    if len(dataSources) == 0 {
        return fmt.Errorf("no enabled data sources found")
    }
    
    // Track overall progress
    var totalItems, processedItems, successCount, errorCount int32
    progressByType := make(map[string]interface{})
    recentErrors := []string{}
    
    // Process each data source
    for i, ds := range dataSources {
        // Check if cancelled
        if jc.IsCancelled() {
            return fmt.Errorf("sync cancelled by user")
        }
        
        // Update progress
        jc.UpdateProgress(map[string]interface{}{
            "current_operation": fmt.Sprintf("Syncing %s data", ds.DataType),
            "percent_complete":  float32(i) / float32(len(dataSources)) * 100,
            "total_items":       totalItems,
            "processed_items":   processedItems,
            "success_count":     successCount,
            "error_count":       errorCount,
            "types":             progressByType,
        })
        
        // Build query
        query := fmt.Sprintf("SELECT * FROM %s", ds.GetFullTableName())
        
        if req.Filter != "" {
            query += fmt.Sprintf(" WHERE %s", req.Filter)
        }
        
        // Add limit if specified
        limit := req.MaxItems
        if limit <= 0 || limit > 10000 {
            limit = 1000 // Default limit per type
        }
        query += fmt.Sprintf(" LIMIT %d", limit)
        
        // Execute query
        rows, err := s.databricks.ExecuteQuery(context.Background(), query)
        if err != nil {
            errorCount++
            errMsg := fmt.Sprintf("Failed to query %s: %v", ds.DataType, err)
            recentErrors = append(recentErrors, errMsg)
            continue
        }
        
        totalItems += int32(len(rows))
        typeSuccess := 0
        typeErrors := 0
        
        // Process in batches
        batchSize := 50
        for j := 0; j < len(rows); j += batchSize {
            if jc.IsCancelled() {
                return fmt.Errorf("sync cancelled by user")
            }
            
            end := j + batchSize
            if end > len(rows) {
                end = len(rows)
            }
            
            batch := rows[j:end]
            succeeded, failed, errors := s.processIngestionBatch(context.Background(), batch, ds.DataType, &ds)
            
            processedItems += int32(len(batch))
            successCount += int32(succeeded)
            errorCount += int32(failed)
            typeSuccess += succeeded
            typeErrors += failed
            
            // Keep only recent errors
            for _, err := range errors {
                if len(recentErrors) >= 10 {
                    recentErrors = recentErrors[1:]
                }
                recentErrors = append(recentErrors, err)
            }
            
            // Update progress
            jc.UpdateProgress(map[string]interface{}{
                "processed_items": processedItems,
                "success_count":   successCount,
                "error_count":     errorCount,
                "recent_errors":   recentErrors,
            })
        }
        
        // Update type progress
        progressByType[ds.DataType] = map[string]interface{}{
            "total":    len(rows),
            "success":  typeSuccess,
            "errors":   typeErrors,
        }
        
        // Update data source stats
        ds.ItemCount = typeSuccess
        ds.LastSyncTime = &[]time.Time{time.Now()}[0]
        ds.LastSyncStatus = "completed"
        s.db.Save(&ds)
    }
    
    // Final progress update
    jc.UpdateProgress(map[string]interface{}{
        "current_operation": "Sync completed",
        "percent_complete":  100,
        "total_items":       totalItems,
        "processed_items":   processedItems,
        "success_count":     successCount,
        "error_count":       errorCount,
        "types":             progressByType,
        "recent_errors":     recentErrors,
    })
    
    log.Printf("Sync job completed: %d/%d items succeeded", successCount, totalItems)
    return nil
}

// executeQueryJob executes a custom query job
func (s *Server) executeQueryJob(jc job.JobContext, req *pb.BLADEQueryJobRequest) error {
    log.Printf("Executing query job: %s", jc.JobID())
    
    // Update initial progress
    jc.UpdateProgress(map[string]interface{}{
        "current_operation": "Executing query",
        "percent_complete":  0,
    })
    
    // Get data source for warehouse info
    var ds datasource.DataSource
    if err := s.db.Where("data_type = ? AND enabled = ?", req.DataType, true).First(&ds).Error; err != nil {
        return fmt.Errorf("no enabled data source found for type %s", req.DataType)
    }
    
    // Execute the query
    rows, err := s.databricks.ExecuteQuery(context.Background(), req.SqlQuery)
    if err != nil {
        return fmt.Errorf("query execution failed: %w", err)
    }
    
    totalItems := int32(len(rows))
    
    // Update progress
    jc.UpdateProgress(map[string]interface{}{
        "current_operation": "Processing query results",
        "percent_complete":  25,
        "total_items":       totalItems,
    })
    
    // Extract catalog config
    catalogConfig := make(map[string]interface{})
    if req.CatalogConfig != nil {
        catalogConfig = req.CatalogConfig.AsMap()
    }
    
    // Process results
    var processedItems, successCount, errorCount int32
    recentErrors := []string{}
    
    batchSize := 100
    for i := 0; i < len(rows); i += batchSize {
        if jc.IsCancelled() {
            return fmt.Errorf("query job cancelled by user")
        }
        
        end := i + batchSize
        if end > len(rows) {
            end = len(rows)
        }
        
        batch := rows[i:end]
        
        // Process each item in batch
        for _, row := range batch {
            processedItems++
            
            // Transform to BLADE item
            item, err := TransformToBLADEItem(req.DataType, row)
            if err != nil {
                errorCount++
                if len(recentErrors) < 10 {
                    recentErrors = append(recentErrors, fmt.Sprintf("Transform error: %v", err))
                }
                continue
            }
            
            // Apply catalog config
            if classification, ok := catalogConfig["classification"].(string); ok {
                item.ClassificationMarking = classification
            }
            
            // Add custom metadata
            metadata := make(map[string]interface{})
            if len(item.Metadata) > 0 {
                json.Unmarshal(item.Metadata, &metadata)
            }
            metadata["query_job_id"] = jc.JobID()
            metadata["query_time"] = time.Now().Format(time.RFC3339)
            
            if customMeta, ok := catalogConfig["metadata"].(map[string]interface{}); ok {
                for k, v := range customMeta {
                    metadata[k] = v
                }
            }
            
            metaJSON, _ := json.Marshal(metadata)
            item.Metadata = metaJSON
            
            // Save and upload
            item.DataSourceID = ds.ID
            if err := s.db.Create(item).Error; err != nil {
                errorCount++
                if len(recentErrors) < 10 {
                    recentErrors = append(recentErrors, fmt.Sprintf("Save error: %v", err))
                }
                continue
            }
            
            if err := s.catalogClient.UploadItem(item); err != nil {
                errorCount++
                if len(recentErrors) < 10 {
                    recentErrors = append(recentErrors, fmt.Sprintf("Upload error: %v", err))
                }
                continue
            }
            
            // Update catalog info
            now := time.Now()
            item.CatalogID = fmt.Sprintf("BLADE-%s-%s", req.DataType, item.ItemID)
            item.UploadedAt = &now
            s.db.Save(item)
            
            successCount++
        }
        
        // Update progress
        percentComplete := float32(processedItems) / float32(totalItems) * 75 + 25
        jc.UpdateProgress(map[string]interface{}{
            "current_operation": fmt.Sprintf("Processing items (%d/%d)", processedItems, totalItems),
            "percent_complete":  percentComplete,
            "processed_items":   processedItems,
            "success_count":     successCount,
            "error_count":       errorCount,
            "recent_errors":     recentErrors,
        })
    }
    
    // Final update
    jc.UpdateProgress(map[string]interface{}{
        "current_operation": "Query job completed",
        "percent_complete":  100,
        "total_items":       totalItems,
        "processed_items":   processedItems,
        "success_count":     successCount,
        "error_count":       errorCount,
        "recent_errors":     recentErrors,
    })
    
    log.Printf("Query job completed: %d/%d items succeeded", successCount, totalItems)
    return nil
}

// ============= Helper Methods =============

func getStringFromProgress(progress map[string]interface{}, key string) string {
    if val, ok := progress[key].(string); ok {
        return val
    }
    return ""
}

func getInt32FromProgress(progress map[string]interface{}, key string) int32 {
    if val, ok := progress[key].(int32); ok {
        return val
    }
    if val, ok := progress[key].(int); ok {
        return int32(val)
    }
    if val, ok := progress[key].(float64); ok {
        return int32(val)
    }
    return 0
}

func getFloatFromProgress(progress map[string]interface{}, key string) float32 {
    if val, ok := progress[key].(float32); ok {
        return val
    }
    if val, ok := progress[key].(float64); ok {
        return float32(val)
    }
    return 0
}

func getErrorsFromProgress(progress map[string]interface{}) []string {
    if errors, ok := progress["recent_errors"].([]string); ok {
        return errors
    }
    if errors, ok := progress["recent_errors"].([]interface{}); ok {
        result := make([]string, 0, len(errors))
        for _, e := range errors {
            if s, ok := e.(string); ok {
                result = append(result, s)
            }
        }
        return result
    }
    return []string{}
}

func estimateCompletion(startTime time.Time, progress map[string]interface{}) *timestamppb.Timestamp {
    percentComplete := getFloatFromProgress(progress, "percent_complete")
    if percentComplete <= 0 {
        return nil
    }
    
    elapsed := time.Since(startTime)
    totalEstimated := time.Duration(float64(elapsed) / float64(percentComplete) * 100)
    remaining := totalEstimated - elapsed
    
    estimatedTime := time.Now().Add(remaining)
    return timestamppb.New(estimatedTime)
}
```

## Key Features of Job Management

### 1. **Sync Jobs**
- Full, incremental, or data-type specific sync
- Progress tracking with type breakdown
- Configurable limits and filters
- Updates data source statistics

### 2. **Query Jobs**
- Execute custom SQL queries
- Transform and ingest results
- Apply custom classification and metadata
- Progress tracking with detailed status

### 3. **Job Control**
- Start/stop operations
- Status monitoring
- Progress estimation
- Cancellation support

### 4. **Progress Tracking**
- Real-time progress updates
- Operation descriptions
- Error collection
- Completion estimation

### 5. **Error Handling**
- Recent error tracking
- Error aggregation
- Graceful failure handling
- Detailed error reporting

## Next Steps

✅ Main server setup complete  
✅ Server struct implemented  
✅ Ingestion endpoints implemented  
✅ Job management endpoints implemented  
➡️ Continue to [04-job-system/01-job-runner.md](../04-job-system/01-job-runner.md) to implement the job runner