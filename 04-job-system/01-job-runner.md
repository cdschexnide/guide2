# Step 1: Job Runner Implementation

This guide covers implementing the job runner system for async operations.

## Create server/job/job.go

Create the main job runner that manages background jobs:

```go
package job

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
    
    "gorm.io/gorm"
)

// JobStatus represents the status of a job
type JobStatus string

const (
    JobStatusPending   JobStatus = "pending"
    JobStatusRunning   JobStatus = "running"
    JobStatusCompleted JobStatus = "completed"
    JobStatusFailed    JobStatus = "failed"
    JobStatusCancelled JobStatus = "cancelled"
    JobStatusPaused    JobStatus = "paused"
)

// JobContext provides context for job execution
type JobContext interface {
    // JobID returns the unique job ID
    JobID() string
    
    // IsCancelled checks if the job has been cancelled
    IsCancelled() bool
    
    // UpdateProgress updates the job progress
    UpdateProgress(progress map[string]interface{})
    
    // GetConfig returns the job configuration
    GetConfig() map[string]interface{}
    
    // SetStatus sets the job status
    SetStatus(status JobStatus)
}

// JobFunction is the function signature for jobs
type JobFunction func(ctx JobContext) error

// JobInfo contains information about a job
type JobInfo struct {
    ID          string
    Type        string
    Status      string
    StartTime   time.Time
    EndTime     *time.Time
    Config      map[string]interface{}
    Progress    map[string]interface{}
    Error       string
}

// JobRunner manages background jobs
type JobRunner struct {
    db          *gorm.DB
    jobs        map[string]*runningJob
    jobsMutex   sync.RWMutex
    workers     int
    workQueue   chan *runningJob
    shutdownCh  chan struct{}
    wg          sync.WaitGroup
}

// runningJob represents a job being executed
type runningJob struct {
    info        JobInfo
    fn          JobFunction
    cancelFn    context.CancelFunc
    ctx         context.Context
    progressMux sync.RWMutex
    statusMux   sync.RWMutex
}

// NewJobRunner creates a new job runner
func NewJobRunner(db *gorm.DB, workers int) *JobRunner {
    if workers <= 0 {
        workers = 5
    }
    
    jr := &JobRunner{
        db:         db,
        jobs:       make(map[string]*runningJob),
        workers:    workers,
        workQueue:  make(chan *runningJob, workers*2),
        shutdownCh: make(chan struct{}),
    }
    
    // Start workers
    for i := 0; i < workers; i++ {
        jr.wg.Add(1)
        go jr.worker(i)
    }
    
    // Start job recovery
    go jr.recoverJobs()
    
    return jr
}

// worker processes jobs from the queue
func (jr *JobRunner) worker(id int) {
    defer jr.wg.Done()
    
    log.Printf("Job worker %d started", id)
    
    for {
        select {
        case job := <-jr.workQueue:
            jr.executeJob(job)
            
        case <-jr.shutdownCh:
            log.Printf("Job worker %d shutting down", id)
            return
        }
    }
}

// executeJob executes a single job
func (jr *JobRunner) executeJob(job *runningJob) {
    log.Printf("Executing job %s (type: %s)", job.info.ID, job.info.Type)
    
    // Update status to running
    job.SetStatus(JobStatusRunning)
    jr.updateJobInDB(job)
    
    // Execute the job function
    err := job.fn(job)
    
    // Update final status
    if err != nil {
        if job.ctx.Err() == context.Canceled {
            job.SetStatus(JobStatusCancelled)
        } else {
            job.SetStatus(JobStatusFailed)
            job.info.Error = err.Error()
        }
    } else {
        job.SetStatus(JobStatusCompleted)
    }
    
    // Set end time
    now := time.Now()
    job.info.EndTime = &now
    
    // Update in database
    jr.updateJobInDB(job)
    
    // Remove from active jobs
    jr.jobsMutex.Lock()
    delete(jr.jobs, job.info.ID)
    jr.jobsMutex.Unlock()
    
    log.Printf("Job %s completed with status: %s", job.info.ID, job.info.Status)
}

// ScheduleJob schedules a new job for execution
func (jr *JobRunner) ScheduleJob(id string, jobType string, fn JobFunction, config map[string]interface{}) error {
    jr.jobsMutex.Lock()
    defer jr.jobsMutex.Unlock()
    
    // Check if job already exists
    if _, exists := jr.jobs[id]; exists {
        return fmt.Errorf("job with ID %s already exists", id)
    }
    
    // Create context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    
    // Create job
    job := &runningJob{
        info: JobInfo{
            ID:        id,
            Type:      jobType,
            Status:    string(JobStatusPending),
            StartTime: time.Now(),
            Config:    config,
            Progress:  make(map[string]interface{}),
        },
        fn:       fn,
        cancelFn: cancel,
        ctx:      ctx,
    }
    
    // Store in map
    jr.jobs[id] = job
    
    // Save to database
    if err := jr.saveJobToDB(job); err != nil {
        delete(jr.jobs, id)
        cancel()
        return fmt.Errorf("failed to save job to database: %w", err)
    }
    
    // Queue for execution
    select {
    case jr.workQueue <- job:
        log.Printf("Job %s scheduled for execution", id)
    default:
        // Queue is full, execute in new goroutine
        jr.wg.Add(1)
        go func() {
            defer jr.wg.Done()
            jr.executeJob(job)
        }()
    }
    
    return nil
}

// CancelJob cancels a running job
func (jr *JobRunner) CancelJob(id string) error {
    jr.jobsMutex.RLock()
    job, exists := jr.jobs[id]
    jr.jobsMutex.RUnlock()
    
    if !exists {
        return fmt.Errorf("job %s not found", id)
    }
    
    // Cancel the job
    job.cancelFn()
    
    log.Printf("Job %s cancelled", id)
    return nil
}

// GetJobInfo returns information about a job
func (jr *JobRunner) GetJobInfo(id string) (JobInfo, error) {
    jr.jobsMutex.RLock()
    job, exists := jr.jobs[id]
    jr.jobsMutex.RUnlock()
    
    if exists {
        job.progressMux.RLock()
        info := job.info
        // Deep copy progress
        info.Progress = make(map[string]interface{})
        for k, v := range job.info.Progress {
            info.Progress[k] = v
        }
        job.progressMux.RUnlock()
        return info, nil
    }
    
    // Check database for completed jobs
    var dbJob JobRecord
    if err := jr.db.Where("job_id = ?", id).First(&dbJob).Error; err != nil {
        return JobInfo{}, fmt.Errorf("job not found")
    }
    
    return jr.jobRecordToInfo(dbJob), nil
}

// GetActiveJobs returns all active jobs
func (jr *JobRunner) GetActiveJobs() []JobInfo {
    jr.jobsMutex.RLock()
    defer jr.jobsMutex.RUnlock()
    
    jobs := make([]JobInfo, 0, len(jr.jobs))
    for _, job := range jr.jobs {
        job.progressMux.RLock()
        info := job.info
        // Deep copy progress
        info.Progress = make(map[string]interface{})
        for k, v := range job.info.Progress {
            info.Progress[k] = v
        }
        job.progressMux.RUnlock()
        jobs = append(jobs, info)
    }
    
    return jobs
}

// GetJobsByType returns jobs of a specific type
func (jr *JobRunner) GetJobsByType(jobType string) []JobInfo {
    // Get active jobs
    activeJobs := jr.GetActiveJobs()
    
    // Get recent completed jobs from DB
    var dbJobs []JobRecord
    jr.db.Where("job_type = ?", jobType).
        Order("start_time DESC").
        Limit(10).
        Find(&dbJobs)
    
    // Combine results
    jobs := make([]JobInfo, 0, len(activeJobs)+len(dbJobs))
    
    // Add active jobs
    for _, job := range activeJobs {
        if job.Type == jobType {
            jobs = append(jobs, job)
        }
    }
    
    // Add DB jobs
    for _, dbJob := range dbJobs {
        // Skip if already in active jobs
        found := false
        for _, active := range activeJobs {
            if active.ID == dbJob.JobID {
                found = true
                break
            }
        }
        if !found {
            jobs = append(jobs, jr.jobRecordToInfo(dbJob))
        }
    }
    
    return jobs
}

// GetJobProgress returns the progress of a job
func (jr *JobRunner) GetJobProgress(id string) map[string]interface{} {
    jr.jobsMutex.RLock()
    job, exists := jr.jobs[id]
    jr.jobsMutex.RUnlock()
    
    if !exists {
        return nil
    }
    
    job.progressMux.RLock()
    defer job.progressMux.RUnlock()
    
    // Deep copy progress
    progress := make(map[string]interface{})
    for k, v := range job.info.Progress {
        progress[k] = v
    }
    
    return progress
}

// HasActiveJobOfType checks if there's an active job of a specific type
func (jr *JobRunner) HasActiveJobOfType(jobType string) bool {
    jr.jobsMutex.RLock()
    defer jr.jobsMutex.RUnlock()
    
    for _, job := range jr.jobs {
        if job.info.Type == jobType && 
           (job.info.Status == string(JobStatusRunning) || 
            job.info.Status == string(JobStatusPending)) {
            return true
        }
    }
    
    return false
}

// ActiveJobs returns the number of active jobs
func (jr *JobRunner) ActiveJobs() int {
    jr.jobsMutex.RLock()
    defer jr.jobsMutex.RUnlock()
    
    count := 0
    for _, job := range jr.jobs {
        if job.info.Status == string(JobStatusRunning) || 
           job.info.Status == string(JobStatusPending) {
            count++
        }
    }
    
    return count
}

// Shutdown gracefully shuts down the job runner
func (jr *JobRunner) Shutdown() {
    log.Println("Shutting down job runner...")
    
    // Signal shutdown
    close(jr.shutdownCh)
    
    // Cancel all running jobs
    jr.jobsMutex.Lock()
    for _, job := range jr.jobs {
        job.cancelFn()
    }
    jr.jobsMutex.Unlock()
    
    // Wait for workers to finish
    jr.wg.Wait()
    
    log.Println("Job runner shutdown complete")
}

// recoverJobs recovers jobs that were running when the server shut down
func (jr *JobRunner) recoverJobs() {
    log.Println("Recovering interrupted jobs...")
    
    var jobs []JobRecord
    jr.db.Where("status IN ?", []string{
        string(JobStatusRunning),
        string(JobStatusPending),
    }).Find(&jobs)
    
    for _, job := range jobs {
        log.Printf("Marking job %s as failed (interrupted)", job.JobID)
        
        job.Status = string(JobStatusFailed)
        job.Error = "Job interrupted by server shutdown"
        now := time.Now()
        job.EndTime = &now
        
        jr.db.Save(&job)
    }
    
    log.Printf("Recovered %d interrupted jobs", len(jobs))
}

// ============= JobContext Implementation =============

func (j *runningJob) JobID() string {
    return j.info.ID
}

func (j *runningJob) IsCancelled() bool {
    select {
    case <-j.ctx.Done():
        return true
    default:
        return false
    }
}

func (j *runningJob) UpdateProgress(progress map[string]interface{}) {
    j.progressMux.Lock()
    defer j.progressMux.Unlock()
    
    // Update progress map
    for k, v := range progress {
        j.info.Progress[k] = v
    }
}

func (j *runningJob) GetConfig() map[string]interface{} {
    return j.info.Config
}

func (j *runningJob) SetStatus(status JobStatus) {
    j.statusMux.Lock()
    defer j.statusMux.Unlock()
    
    j.info.Status = string(status)
}
```

## Create Database Models for Jobs

Create `server/job/models.go`:

```go
package job

import (
    "encoding/json"
    "time"
    
    "gorm.io/gorm"
    "gorm.io/datatypes"
)

// JobRecord represents a job in the database
type JobRecord struct {
    gorm.Model
    JobID      string         `gorm:"uniqueIndex;not null"`
    JobType    string         `gorm:"index;not null"`
    Status     string         `gorm:"index;not null"`
    StartTime  time.Time      `gorm:"not null"`
    EndTime    *time.Time
    Config     datatypes.JSON
    Progress   datatypes.JSON
    Error      string
}

// TableName specifies the table name
func (JobRecord) TableName() string {
    return "jobs"
}

// saveJobToDB saves a job to the database
func (jr *JobRunner) saveJobToDB(job *runningJob) error {
    configJSON, err := json.Marshal(job.info.Config)
    if err != nil {
        return err
    }
    
    progressJSON, err := json.Marshal(job.info.Progress)
    if err != nil {
        return err
    }
    
    record := JobRecord{
        JobID:     job.info.ID,
        JobType:   job.info.Type,
        Status:    job.info.Status,
        StartTime: job.info.StartTime,
        EndTime:   job.info.EndTime,
        Config:    configJSON,
        Progress:  progressJSON,
        Error:     job.info.Error,
    }
    
    return jr.db.Create(&record).Error
}

// updateJobInDB updates a job in the database
func (jr *JobRunner) updateJobInDB(job *runningJob) error {
    job.progressMux.RLock()
    progressJSON, err := json.Marshal(job.info.Progress)
    job.progressMux.RUnlock()
    
    if err != nil {
        return err
    }
    
    updates := map[string]interface{}{
        "status":   job.info.Status,
        "progress": progressJSON,
        "error":    job.info.Error,
    }
    
    if job.info.EndTime != nil {
        updates["end_time"] = job.info.EndTime
    }
    
    return jr.db.Model(&JobRecord{}).
        Where("job_id = ?", job.info.ID).
        Updates(updates).Error
}

// jobRecordToInfo converts a database record to JobInfo
func (jr *JobRunner) jobRecordToInfo(record JobRecord) JobInfo {
    info := JobInfo{
        ID:        record.JobID,
        Type:      record.JobType,
        Status:    record.Status,
        StartTime: record.StartTime,
        EndTime:   record.EndTime,
        Error:     record.Error,
    }
    
    // Unmarshal config
    if len(record.Config) > 0 {
        json.Unmarshal(record.Config, &info.Config)
    }
    if info.Config == nil {
        info.Config = make(map[string]interface{})
    }
    
    // Unmarshal progress
    if len(record.Progress) > 0 {
        json.Unmarshal(record.Progress, &info.Progress)
    }
    if info.Progress == nil {
        info.Progress = make(map[string]interface{})
    }
    
    return info
}
```

## Key Features of Job Runner

### 1. **Worker Pool**
- Configurable number of workers
- Queue-based job distribution
- Graceful overflow handling

### 2. **Job Lifecycle**
- Pending → Running → Completed/Failed/Cancelled
- Progress tracking during execution
- Error capture and reporting

### 3. **Persistence**
- Jobs saved to database
- Progress updates persisted
- Recovery after server restart

### 4. **Job Control**
- Cancellation support
- Status monitoring
- Type-based queries

### 5. **Thread Safety**
- Concurrent access protection
- Safe progress updates
- Atomic state transitions

## Testing the Job Runner

Create a simple test:

```go
// server/job/job_test.go
package job

import (
    "fmt"
    "testing"
    "time"
)

func TestJobRunner(t *testing.T) {
    // This is a basic test structure
    // In production, you'd use a test database
    
    t.Run("job execution", func(t *testing.T) {
        // Test job execution flow
        jobFunc := func(ctx JobContext) error {
            // Simulate work
            for i := 0; i <= 100; i += 10 {
                if ctx.IsCancelled() {
                    return fmt.Errorf("job cancelled")
                }
                
                ctx.UpdateProgress(map[string]interface{}{
                    "percent": i,
                })
                
                time.Sleep(10 * time.Millisecond)
            }
            
            return nil
        }
        
        // Would test with actual JobRunner instance
        t.Skip("Requires database setup")
    })
}
```

## Integration with Server

The job runner is already integrated in the main server through:
- `main.go`: Creates and passes JobRunner to server
- `server.go`: Uses JobRunner for async operations
- Job endpoints use the runner for scheduling

## Next Steps

✅ Job runner implementation complete  
✅ Database models for jobs created  
✅ Thread-safe execution model  
➡️ Continue to [05-database/01-database-setup.md](../05-database/01-database-setup.md) to set up the database