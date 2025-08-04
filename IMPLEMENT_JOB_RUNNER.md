# Implementing Job Runner Framework in Minimal BLADE Service

This guide provides step-by-step instructions to implement the job runner framework exactly as it exists in infinityai-cataloger. This implementation will add scheduled job capabilities with pause/resume/kill functionality to your BLADE service.

## Prerequisites

Ensure you have completed the `RESTORE_CATALOG_UPLOAD.md` implementation before proceeding, as this builds on that foundation.

## Overview

The job runner framework provides:
- **Scheduled Job Execution**: Jobs can be scheduled to run at specific times
- **Job Control**: Pause, resume, and kill running jobs
- **Job Status Tracking**: Monitor job progress and status
- **Job History**: Track all executed jobs
- **Context-Aware Jobs**: Jobs receive database context and can check for stop signals

## Dependencies

Add the following dependency to your `go.mod`:

```go
require (
    github.com/go-co-op/gocron/v2 v2.16.3
)
```

## Implementation Steps

### Step 1: Create Core Job Framework Files

Create the following directory structure:

```
server/
├── blade_server/
│   ├── server.go
│   └── catalog_upload.go
├── job/                    # New directory
│   ├── job.go
│   ├── job_context.go
│   ├── job_control.go
│   ├── job_runner.go
│   └── job_status.go
└── main.go
```

#### 1A. Create `server/job/job_status.go`

```go
package job

type JobStatus int

const (
	Pending    JobStatus = iota // Started Job in the future
	InProgress                  // Job currently running
	Paused                      // Manually paused a running job
	Passed                      // Job was successful
	Failed                      // Job failed without user interaction
	Cancelled                   // Manually killed a pending job
	Stopped                     // Manually killed a running job
)

func (s JobStatus) String() string {
	switch s {
	case Pending:
		return "Pending"
	case InProgress:
		return "In Progress"
	case Paused:
		return "Paused"
	case Passed:
		return "Passed"
	case Failed:
		return "Failed"
	case Cancelled:
		return "Cancelled"
	case Stopped:
		return "Stopped"
	default:
		return "Unknown"
	}
}

func (s JobStatus) IsCompleted() bool {
	return s == Passed || s == Failed || s == Stopped || s == Cancelled
}
```

#### 1B. Create `server/job/job_context.go`

```go
package job

import (
	"context"

	"gorm.io/gorm"
)

type JobContext struct {
	Ctx                   context.Context
	ProcessControlSignals func(context.Context, func(JobStatus, string)) bool
	UpdateStatus          func(JobStatus, string)
	DB                    *gorm.DB
}

func (jc JobContext) ShouldStop() bool {
	return jc.ProcessControlSignals(jc.Ctx, jc.UpdateStatus)
}
```

#### 1C. Create `server/job/job_control.go`

```go
package job

import (
	"context"
	"fmt"
)

type JobControl struct {
	Paused   bool
	PauseCh  chan struct{}
	ResumeCh chan struct{}
	KillCh   chan struct{}
}

func NewJobControl() *JobControl {
	return &JobControl{
		PauseCh:  make(chan struct{}, 1),
		ResumeCh: make(chan struct{}, 1),
		KillCh:   make(chan struct{}, 1),
	}
}

func (jc *JobControl) ProcessControlSignals(ctx context.Context, updateStatusFn func(JobStatus, string)) bool {
	if jc.checkInterrupt(ctx, updateStatusFn) {
		return true
	}

	// if the job was killed or ran out of time, end early
	if jc.handlePause(ctx, updateStatusFn) {
		return true
	}
	return false
}

func (jc *JobControl) checkInterrupt(ctx context.Context, updateStatusFn func(JobStatus, string)) bool {
	select {
	case <-jc.KillCh:
		updateStatusFn(Stopped, "Job Stopped")
		return true
	case <-ctx.Done():
		updateStatusFn(Failed, "Job Timed Out")
		return true
	default:
		return false
	}
}

// Returns true if job should be killed, false otherwise
func (jc *JobControl) handlePause(ctx context.Context, updateStatusFn func(JobStatus, string)) bool {
	// Check if a new pause signal arrived (non-blocking)
	select {
	case <-jc.PauseCh:
		if !jc.Paused {
			jc.Paused = true
			updateStatusFn(Paused, "Paused")
		}
	default:
		// No new pause signal
	}

	if !jc.Paused {
		return false
	}

	// Block waiting for resume, kill, or context done
	for {
		fmt.Println("Waiting for resume or kill signal")
		select {
		case <-jc.ResumeCh:
			jc.Paused = false
			updateStatusFn(InProgress, "Resumed")
			return false // continue processing
		case <-jc.KillCh:
			updateStatusFn(Stopped, "Stopped while paused")
			return true // must stop
		case <-ctx.Done():
			updateStatusFn(Failed, "Timed out while paused")
			return true // must stop
		}
	}
}

func (jc *JobControl) Kill() error {
	select {
	case jc.KillCh <- struct{}{}:
		// signal sent
	default:
		// channel already full, do nothing
	}
	return nil
}

func (jc *JobControl) Pause() error {
	// Job is already paused
	if jc.Paused {
		return nil
	}

	select {
	case jc.PauseCh <- struct{}{}:
		// signal sent
	default:
		// channel already full, do nothing
	}
	return nil
}

func (jc *JobControl) Resume() error {
	// Job is not paused
	if !jc.Paused {
		return nil
	}

	select {
	case jc.ResumeCh <- struct{}{}:
		// signal sent
	default:
		// channel already full, do nothing
	}
	return nil
}
```

#### 1D. Create `server/job/job.go`

```go
package job

import (
	"context"
	"fmt"
	"time"

	"github.com/go-co-op/gocron/v2"
	"github.com/google/uuid"
)

type RunnableJob interface {
	Schedule(params ScheduleJobParams) error
	Kill() error
	Pause() error
	Resume() error
	Status() JobStatus
	ScheduledStartTime() *time.Time
	StartTime() *time.Time
	StopTime() *time.Time
	TotalRunTime() time.Duration
	MaxRunTime() *time.Duration
	SetOnStatusChange(func(status JobStatus, reason string))
}

type ScheduledJob struct {
	ID                 string
	status             JobStatus
	scheduledStartTime *time.Time
	startTime          *time.Time
	stopTime           *time.Time
	maxRunTime         *time.Duration
	onStatusChange     func(status JobStatus, reason string)
	scheduler          gocron.Scheduler
	Run                JobFunction
	*JobControl
}

type JobFunction func(jc JobContext) error

var NewJob = NewJobDefault

func NewJobDefault(exec JobFunction) RunnableJob {
	scheduler, _ := gocron.NewScheduler(gocron.WithLocation(time.UTC), gocron.WithStopTimeout(5*time.Second))
	scheduler.Start()
	return &ScheduledJob{
		ID:         uuid.NewString(),
		status:     Pending,
		scheduler:  scheduler,
		JobControl: NewJobControl(),
		Run:        exec,
	}

}

func (j *ScheduledJob) Schedule(params ScheduleJobParams) error {
	fmt.Printf("Schedule received: %v\n", params)
	j.updateStatus(Pending, "Job Created")
	j.scheduledStartTime = &params.StartTime
	_, err := j.scheduler.NewJob(
		gocron.OneTimeJob(gocron.OneTimeJobStartDateTime(params.StartTime.UTC())),

		gocron.NewTask(func() {
			// TODO: Should we have a -1 flag to denote that this task doesn't have a MaximumExecutionTime
			ctx, cancel := context.WithTimeout(context.Background(),
				time.Duration(params.MaximumExecutionTimeInMs)*time.Millisecond)
			defer cancel()

			j.updateStatus(InProgress, "Job Started")
			nowTimestamp := time.Now().UTC()
			j.startTime = &nowTimestamp
			duration := time.Duration(params.MaximumExecutionTimeInMs) * time.Millisecond
			j.maxRunTime = &duration

			j.Run(JobContext{
				Ctx:                   ctx,
				ProcessControlSignals: j.ProcessControlSignals,
				UpdateStatus:          j.updateStatus,
				DB:                    params.Database,
			})

			j.updateStatus(Passed, "Job Completed")
		}),
		gocron.WithSingletonMode(gocron.LimitModeReschedule),
	)
	if err != nil {
		fmt.Printf("Err: %v", err)
		return err
	}

	return nil
}

func (j *ScheduledJob) Status() JobStatus {
	return j.status
}

func (j *ScheduledJob) ScheduledStartTime() *time.Time {
	return j.scheduledStartTime
}

func (j *ScheduledJob) StartTime() *time.Time {
	return j.startTime
}

func (j *ScheduledJob) StopTime() *time.Time {
	return j.stopTime
}

func (j *ScheduledJob) TotalRunTime() time.Duration {
	if j.startTime == nil {
		return 0
	}
	return time.Since(*j.startTime)
}

func (j *ScheduledJob) MaxRunTime() *time.Duration {
	return j.maxRunTime
}

func (j *ScheduledJob) SetOnStatusChange(f func(status JobStatus, reason string)) {
	// the job's private/internal onStatusChange method is set to the parent's OnStatusChange listener
	j.onStatusChange = f
}

func (j *ScheduledJob) updateStatus(s JobStatus, reason string) {
	j.status = s
	// If the parent has set the OnStatusChange listener, notify the parent of the status update
	if j.onStatusChange != nil {
		j.onStatusChange(s, reason)
	}

	if j.status.IsCompleted() {
		nowTimestamp := time.Now().UTC()
		j.stopTime = &nowTimestamp
	}
}
```

#### 1E. Create `server/job/job_runner.go`

```go
package job

import (
	"errors"
	"fmt"
	"sync"
	"time"

	"gorm.io/gorm"
)

type ScheduleJobParams struct {
	JobFunction              JobFunction
	StartTime                time.Time
	MaximumExecutionTimeInMs int
	Database                 *gorm.DB
	// TODO: Potentially Add a ScheduledEndTime - a time that denotes when the process will exit regardless of status
}

type KillJobParams struct {
}

type PauseJobParams struct {
}

type ResumeJobParams struct {
}

// Metadata for Jobs
// result
// execution time
// job name
// timestamp for start
// timestamp for end
type JobController interface {
	Schedule(params ScheduleJobParams) error // can schedule rather than starting immediately
	Kill(params KillJobParams) error         // Kills/cancels the job
	Pause(params PauseJobParams) error
	Resume(params ResumeJobParams) error
	CurrentJob() RunnableJob
	JobHistory() []RunnableJob
}

type JobRunner struct {
	mu         sync.Mutex
	currentJob RunnableJob
	jobHistory []RunnableJob
}

func NewJobRunner() JobController {
	return &JobRunner{
		jobHistory: make([]RunnableJob, 0),
	}
}

func (runner *JobRunner) Schedule(params ScheduleJobParams) error {
	runner.mu.Lock()
	if runner.currentJob != nil {
		return errors.New("job already exists")
	}

	if params.JobFunction == nil {
		return errors.New("JobFunction is required")
	}

	job := NewJob(params.JobFunction)
	runner.currentJob = job
	runner.jobHistory = append(runner.jobHistory, job)
	runner.mu.Unlock()

	// Set a listener so that the runner can be notified whenever the job's status changes
	job.SetOnStatusChange(func(status JobStatus, reason string) {
		fmt.Printf("Status Changed: %s, Reason: %s\n", status.String(), reason)
		runner.mu.Lock()
		defer runner.mu.Unlock()

		if status.IsCompleted() {
			runner.currentJob = nil
		}
	})

	fmt.Println("Scheduling job from runner")
	return job.Schedule(params)

}

func (runner *JobRunner) Kill(params KillJobParams) error {
	runner.mu.Lock()
	defer runner.mu.Unlock()

	if runner.currentJob != nil {
		return runner.currentJob.Kill()
	}
	return nil
}

func (runner *JobRunner) Pause(params PauseJobParams) error {
	runner.mu.Lock()
	defer runner.mu.Unlock()

	if runner.currentJob != nil {
		return runner.currentJob.Pause()
	}
	return nil
}

func (runner *JobRunner) Resume(params ResumeJobParams) error {
	runner.mu.Lock()
	defer runner.mu.Unlock()

	if runner.currentJob != nil {
		return runner.currentJob.Resume()
	}
	return nil
}

func (runner *JobRunner) JobHistory() []RunnableJob {
	runner.mu.Lock()
	defer runner.mu.Unlock()
	return runner.jobHistory
}

func (runner *JobRunner) CurrentJob() RunnableJob {
	runner.mu.Lock()
	defer runner.mu.Unlock()

	return runner.currentJob
}
```

### Step 2: Update Server Integration

#### 2A. Update `server/blade_server/server.go`

Add the JobRunner to your server struct and update the import statements:

```go
package blade_server

import (
	"context"
	"log"
	"os"
	"strings"
	"time"

	"minimal-blade-ingestion/database/datasource"
	pb "minimal-blade-ingestion/generated/proto"
	"minimal-blade-ingestion/server/job"  // Add this import

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"
	"gorm.io/gorm"
)

// Server gRPC server and gorm database for the cataloger
type Server struct {
	pb.UnimplementedBLADEMinimalServiceServer
	DB        *gorm.DB
	JobRunner job.JobController  // Add this field
}

// NewServer creates a new BLADE minimal server
func NewServer(db *gorm.DB) *Server {
	return &Server{
		DB:        db,
		JobRunner: job.NewJobRunner(),  // Initialize JobRunner
	}
}

// ... existing methods ...

// ============= Job Control Endpoints =============

// StartBLADEJob starts a BLADE data processing job
func (s *Server) BLADEStartJob(ctx context.Context, req *emptypb.Empty) (*pb.BLADEStartJobResponse, error) {
	log.Println("Starting BLADE job")
	
	err := s.JobRunner.Schedule(job.ScheduleJobParams{
		StartTime:                time.Now().Add(time.Millisecond * 100),
		JobFunction:              s.sampleBLADEJob, // We'll implement this next
		MaximumExecutionTimeInMs: 3600000, // 1 hour
		Database:                 s.DB,
	})
	
	if err != nil {
		log.Printf("Error scheduling job: %v", err)
		return nil, status.Error(codes.Internal, "failed to schedule job")
	}
	
	return &pb.BLADEStartJobResponse{Status: "OK"}, nil
}

// StopBLADEJob stops the current BLADE job
func (s *Server) BLADEStopJob(ctx context.Context, req *emptypb.Empty) (*pb.BLADEStopJobResponse, error) {
	log.Println("Stopping BLADE job")
	
	err := s.JobRunner.Kill(job.KillJobParams{})
	if err != nil {
		log.Printf("Error killing job: %v", err)
		return nil, status.Error(codes.Internal, "failed to stop job")
	}
	
	return &pb.BLADEStopJobResponse{Status: "OK"}, nil
}

// PauseBLADEJob pauses the current BLADE job
func (s *Server) BLADEPauseJob(ctx context.Context, req *emptypb.Empty) (*pb.BLADEPauseJobResponse, error) {
	log.Println("Pausing BLADE job")
	
	err := s.JobRunner.Pause(job.PauseJobParams{})
	if err != nil {
		log.Printf("Error pausing job: %v", err)
		return nil, status.Error(codes.Internal, "failed to pause job")
	}
	
	return &pb.BLADEPauseJobResponse{Status: "OK"}, nil
}

// ResumeBLADEJob resumes the current BLADE job
func (s *Server) BLADEResumeJob(ctx context.Context, req *emptypb.Empty) (*pb.BLADEResumeJobResponse, error) {
	log.Println("Resuming BLADE job")
	
	err := s.JobRunner.Resume(job.ResumeJobParams{})
	if err != nil {
		log.Printf("Error resuming job: %v", err)
		return nil, status.Error(codes.Internal, "failed to resume job")
	}
	
	return &pb.BLADEResumeJobResponse{Status: "OK"}, nil
}

// GetBLADEJobProgress gets the current job progress
func (s *Server) BLADEJobProgress(ctx context.Context, req *emptypb.Empty) (*pb.BLADEJobProgressResponse, error) {
	currentJob := s.JobRunner.CurrentJob()
	jobStatus := "N/A"
	scheduledStartTime := "N/A"
	actualStartTime := "N/A"
	totalRunTime := "N/A"
	maxTime := "N/A"
	
	if currentJob != nil {
		jobStatus = currentJob.Status().String()
		if currentJob.ScheduledStartTime() != nil {
			scheduledStartTime = currentJob.ScheduledStartTime().String()
		}
		if currentJob.StartTime() != nil {
			actualStartTime = currentJob.StartTime().String()
		}
		totalRunTime = currentJob.TotalRunTime().String()
		if currentJob.MaxRunTime() != nil {
			maxTime = currentJob.MaxRunTime().String()
		}
	}

	// Get total processed items from data sources
	var totalSources, processedItems int64
	s.DB.Model(&datasource.DataSource{}).Count(&totalSources)
	// This is a placeholder - you can implement actual progress tracking
	processedItems = 0

	return &pb.BLADEJobProgressResponse{
		Status:             jobStatus,
		ScheduledStartTime: scheduledStartTime,
		ActualStartTime:    actualStartTime,
		MaximumRunTime:     maxTime,
		TotalRunTime:       totalRunTime,
		TotalItems:         int32(totalSources),
		ProcessedItems:     int32(processedItems),
	}, nil
}

// GetBLADEJobHistory gets the job history
func (s *Server) BLADEJobHistory(ctx context.Context, req *emptypb.Empty) (*pb.BLADEJobHistoryResponse, error) {
	jobs := s.JobRunner.JobHistory()

	var jobRecords []*pb.BLADEJobRecord
	for _, job := range jobs {
		var scheduledStartTimeStr, actualStartTimeStr, maxRunTimeStr, stopTimeStr string

		if job.ScheduledStartTime() != nil {
			scheduledStartTimeStr = job.ScheduledStartTime().String()
		}
		if job.StartTime() != nil {
			actualStartTimeStr = job.StartTime().String()
		}
		if job.MaxRunTime() != nil {
			maxRunTimeStr = job.MaxRunTime().String()
		}
		if job.StopTime() != nil {
			stopTimeStr = job.StopTime().String()
		}

		jobRecords = append(jobRecords, &pb.BLADEJobRecord{
			Status:             job.Status().String(),
			ScheduledStartTime: scheduledStartTimeStr,
			ActualStartTime:    actualStartTimeStr,
			MaximumRunTime:     maxRunTimeStr,
			TotalRunTime:       job.TotalRunTime().String(),
			StopTime:           stopTimeStr,
		})
	}

	return &pb.BLADEJobHistoryResponse{
		Jobs: jobRecords,
	}, nil
}

// sampleBLADEJob is an example job function
func (s *Server) sampleBLADEJob(jc job.JobContext) error {
	log.Println("Starting sample BLADE job")
	
	// Get all data sources
	var sources []datasource.DataSource
	if err := jc.DB.Find(&sources).Error; err != nil {
		log.Printf("Error loading data sources: %v", err)
		return err
	}
	
	log.Printf("Processing %d data sources", len(sources))
	
	// Process each data source
	for i, source := range sources {
		// Check if we should stop
		if jc.ShouldStop() {
			log.Println("Job stopping due to control signal")
			return nil
		}
		
		log.Printf("Processing source %d/%d: %s", i+1, len(sources), source.Name)
		
		// Simulate work
		time.Sleep(time.Second * 2)
		
		// Here you would implement actual BLADE data processing
		// For example:
		// - Query BLADE data from the source
		// - Process and transform the data
		// - Upload to catalog (if catalog upload is implemented)
		
		jc.UpdateStatus(job.InProgress, fmt.Sprintf("Processed %d/%d sources", i+1, len(sources)))
	}
	
	log.Println("Sample BLADE job completed successfully")
	return nil
}
```

### Step 3: Update Protocol Buffer Definitions

#### 3A. Update `proto/blade_minimal.proto`

Add the job control endpoints to your service definition:

```protobuf
service BLADEMinimalService {
  // ... existing endpoints ...
  
  // ============= Job Control Endpoints =============
  
  // Start a BLADE data processing job
  rpc BLADEStartJob(google.protobuf.Empty) returns (BLADEStartJobResponse) {
    option (google.api.http) = {
      post: "/blade/job/start"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Start BLADE data processing job";
      description: "Starts a scheduled job to process BLADE data from configured sources.";
    };
  }
  
  // Stop the current BLADE job
  rpc BLADEStopJob(google.protobuf.Empty) returns (BLADEStopJobResponse) {
    option (google.api.http) = {
      post: "/blade/job/stop"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Stop BLADE job";
      description: "Stops the currently running BLADE job.";
    };
  }
  
  // Pause the current BLADE job
  rpc BLADEPauseJob(google.protobuf.Empty) returns (BLADEPauseJobResponse) {
    option (google.api.http) = {
      post: "/blade/job/pause"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Pause BLADE job";
      description: "Pauses the currently running BLADE job.";
    };
  }
  
  // Resume the current BLADE job
  rpc BLADEResumeJob(google.protobuf.Empty) returns (BLADEResumeJobResponse) {
    option (google.api.http) = {
      post: "/blade/job/resume"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Resume BLADE job";
      description: "Resumes a paused BLADE job.";
    };
  }
  
  // Get current job progress
  rpc BLADEJobProgress(google.protobuf.Empty) returns (BLADEJobProgressResponse) {
    option (google.api.http) = {
      get: "/blade/job/progress"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Get job progress";
      description: "Gets the current status and progress of the BLADE job.";
    };
  }
  
  // Get job history
  rpc BLADEJobHistory(google.protobuf.Empty) returns (BLADEJobHistoryResponse) {
    option (google.api.http) = {
      get: "/blade/job/history"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      tags: "Job Control";
      summary: "Get job history";
      description: "Gets the history of all executed BLADE jobs.";
    };
  }
}
```

Add the corresponding message definitions:

```protobuf
// Job Control Messages

message BLADEStartJobResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Job start status"
      example: "\"OK\""
    }];
}

message BLADEStopJobResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Job stop status"
      example: "\"OK\""
    }];
}

message BLADEPauseJobResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Job pause status"
      example: "\"OK\""
    }];
}

message BLADEResumeJobResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Job resume status"
      example: "\"OK\""
    }];
}

message BLADEJobProgressResponse {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Current job status"
      example: "\"In Progress\""
    }];
  
  string scheduledStartTime = 2 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "When the job was scheduled to start"
    }];
  
  string actualStartTime = 3 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "When the job actually started"
    }];
  
  string maximumRunTime = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Maximum allowed run time"
    }];
  
  string totalRunTime = 5 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Total time the job has been running"
    }];
  
  int32 totalItems = 6 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Total number of items to process"
      example: "100"
    }];
  
  int32 processedItems = 7 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Number of items processed so far"
      example: "45"
    }];
}

message BLADEJobHistoryResponse {
  repeated BLADEJobRecord jobs = 1;
}

message BLADEJobRecord {
  string status = 1 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Job completion status"
    }];
  
  string scheduledStartTime = 2 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "When the job was scheduled to start"
    }];
  
  string actualStartTime = 3 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "When the job actually started"
    }];
  
  string maximumRunTime = 4 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Maximum allowed run time"
    }];
  
  string totalRunTime = 5 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "Actual total run time"
    }];
  
  string stopTime = 6 [
    (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
      description: "When the job stopped"
    }];
}
```

### Step 4: Update Dependencies

#### 4A. Update `go.mod`

Add the gocron dependency:

```bash
go get github.com/go-co-op/gocron/v2@v2.16.3
```

Your go.mod should now include:

```go
require (
    // ... existing dependencies ...
    github.com/go-co-op/gocron/v2 v2.16.3
    github.com/google/uuid v1.6.0
)
```

### Step 5: Generate Protocol Buffer Files

After updating the proto file, regenerate the gRPC code:

```bash
make proto
```

This will generate the new job control endpoints in your generated proto files.

### Step 6: Create Example BLADE Job Implementation

#### 6A. Create `server/blade_server/blade_job_example.go`

```go
package blade_server

import (
	"fmt"
	"log"
	"time"

	"minimal-blade-ingestion/database/datasource"
	"minimal-blade-ingestion/server/job"
)

// ExampleBLADEDataProcessor is a more comprehensive example job
func (s *Server) ExampleBLADEDataProcessor(jc job.JobContext) error {
	log.Println("Starting BLADE data processing job")
	
	// Get all configured BLADE data sources
	var sources []datasource.DataSource
	if err := jc.DB.Find(&sources).Error; err != nil {
		log.Printf("Error loading data sources: %v", err)
		return err
	}
	
	log.Printf("Found %d BLADE data sources to process", len(sources))
	jc.UpdateStatus(job.InProgress, fmt.Sprintf("Starting processing of %d sources", len(sources)))
	
	processedCount := 0
	
	for i, source := range sources {
		// Check if we should stop (kill signal, timeout, etc.)
		if jc.ShouldStop() {
			log.Printf("Job stopping after processing %d/%d sources", processedCount, len(sources))
			return nil
		}
		
		log.Printf("Processing source %d/%d: %s (%s)", i+1, len(sources), source.DisplayName, source.Name)
		
		// Simulate processing time for each source
		startTime := time.Now()
		
		// Here you would implement actual BLADE data processing:
		// 1. Connect to the BLADE data source
		// 2. Query for new/updated data
		// 3. Transform the data as needed
		// 4. Upload to catalog (if catalog upload is implemented)
		// 5. Update processing status
		
		// For this example, we'll simulate work with a sleep
		time.Sleep(time.Second * 3)
		
		// Check again if we should stop after processing
		if jc.ShouldStop() {
			log.Printf("Job stopping after processing source: %s", source.Name)
			return nil
		}
		
		processedCount++
		processingTime := time.Since(startTime)
		
		log.Printf("Completed processing source %s in %v", source.Name, processingTime)
		jc.UpdateStatus(job.InProgress, fmt.Sprintf("Processed %d/%d sources (%s completed)", processedCount, len(sources), source.Name))
		
		// Add a small delay between sources to allow for pause/resume testing
		time.Sleep(time.Millisecond * 500)
	}
	
	log.Printf("BLADE data processing job completed successfully. Processed %d sources.", processedCount)
	return nil
}

// QuickBLADEJob is a faster job for testing
func (s *Server) QuickBLADEJob(jc job.JobContext) error {
	log.Println("Starting quick BLADE job")
	
	for i := 0; i < 10; i++ {
		if jc.ShouldStop() {
			log.Printf("Quick job stopping at iteration %d", i)
			return nil
		}
		
		log.Printf("Quick job iteration %d/10", i+1)
		jc.UpdateStatus(job.InProgress, fmt.Sprintf("Iteration %d/10", i+1))
		
		time.Sleep(time.Second * 2)
	}
	
	log.Println("Quick BLADE job completed")
	return nil
}
```

You can modify the `BLADEStartJob` method to use either job function:

```go
// In BLADEStartJob method, replace the JobFunction line with:
JobFunction:              s.ExampleBLADEDataProcessor, // or s.QuickBLADEJob for testing
```

### Step 7: Testing the Implementation

#### 7A. Build and Start Services

```bash
# Build and start all services
docker compose up -d --build
```

#### 7B. Test Job Control Endpoints

```bash
# Start a job
curl -X POST "http://localhost:9091/blade/job/start"

# Check job progress
curl "http://localhost:9091/blade/job/progress"

# Pause the job
curl -X POST "http://localhost:9091/blade/job/pause"

# Resume the job
curl -X POST "http://localhost:9091/blade/job/resume"

# Stop the job
curl -X POST "http://localhost:9091/blade/job/stop"

# View job history
curl "http://localhost:9091/blade/job/history"
```

#### 7C. Expected Responses

**Job Progress Response:**
```json
{
  "status": "In Progress",
  "scheduledStartTime": "2024-01-15T10:30:00Z",
  "actualStartTime": "2024-01-15T10:30:00.100Z",
  "maximumRunTime": "1h0m0s",
  "totalRunTime": "2m30s",
  "totalItems": 3,
  "processedItems": 1
}
```

**Job History Response:**
```json
{
  "jobs": [
    {
      "status": "Passed",
      "scheduledStartTime": "2024-01-15T10:30:00Z",
      "actualStartTime": "2024-01-15T10:30:00.100Z",
      "maximumRunTime": "1h0m0s",
      "totalRunTime": "5m45s",
      "stopTime": "2024-01-15T10:35:45Z"
    }
  ]
}
```

### Step 8: Advanced Usage Patterns

#### 8A. Custom Job Functions

You can create specialized job functions for different types of BLADE processing:

```go
// ProcessMaintenanceData processes maintenance-specific BLADE data
func (s *Server) ProcessMaintenanceData(jc job.JobContext) error {
	// Implementation specific to maintenance data
	return nil
}

// ProcessSortieData processes sortie-specific BLADE data  
func (s *Server) ProcessSortieData(jc job.JobContext) error {
	// Implementation specific to sortie data
	return nil
}
```

#### 8B. Job Scheduling Variations

You can schedule jobs with different parameters:

```go
// Schedule a job to start in 5 minutes with 2-hour timeout
s.JobRunner.Schedule(job.ScheduleJobParams{
	StartTime:                time.Now().Add(5 * time.Minute),
	JobFunction:              s.ExampleBLADEDataProcessor,
	MaximumExecutionTimeInMs: 7200000, // 2 hours
	Database:                 s.DB,
})
```

#### 8C. Job Progress Tracking

For more detailed progress tracking, you can extend the job context:

```go
func (s *Server) DetailedBLADEJob(jc job.JobContext) error {
	totalSteps := 100
	
	for i := 0; i < totalSteps; i++ {
		if jc.ShouldStop() {
			return nil
		}
		
		// Update progress with detailed information
		progressMsg := fmt.Sprintf("Step %d/%d: Processing item %d", i+1, totalSteps, i+1)
		jc.UpdateStatus(job.InProgress, progressMsg)
		
		// Do actual work here
		time.Sleep(time.Millisecond * 100)
	}
	
	return nil
}
```

## Troubleshooting

### Common Issues

1. **Job doesn't start**
   - Check if another job is already running
   - Verify the JobFunction is not nil
   - Check server logs for scheduling errors

2. **Job doesn't respond to pause/resume**
   - Ensure your job function calls `jc.ShouldStop()` regularly
   - Jobs that don't check for control signals won't pause

3. **Job times out unexpectedly**
   - Check the MaximumExecutionTimeInMs parameter
   - Ensure long-running operations periodically check `jc.ShouldStop()`

4. **Missing endpoints**
   - Regenerate proto files with `make proto`
   - Restart the service after updating proto definitions

### Monitoring Job Execution

Check the service logs to monitor job execution:

```bash
docker compose logs minimal-blade-service -f
```

Look for log messages like:
- "Starting BLADE job"
- "Status Changed: In Progress, Reason: Job Started"
- "Processing source X/Y: ..."
- "Job stopping due to control signal"

## Architecture Notes

This implementation follows the exact same patterns as infinityai-cataloger:

- **Separation of Concerns**: Job logic is separate from job control
- **Thread Safety**: JobRunner uses mutexes to handle concurrent access
- **Context Awareness**: Jobs receive context and can respond to control signals  
- **Status Tracking**: Complete job lifecycle tracking with timestamps
- **Graceful Shutdown**: Jobs can be paused, resumed, or killed cleanly

The job runner framework is now fully integrated into your BLADE service with the same capabilities as the infinityai-cataloger implementation.
