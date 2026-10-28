# Industrial Processing System

![CI](https://github.com/marijelaveljkovic/IndustrialProcessingSystem/actions/workflows/ci.yml/badge.svg)

A multithreaded job processing service in C# (.NET 8) that schedules and executes jobs by priority, with retry logic, event-based logging and periodic reporting.

Developed individually as a university project at the Faculty of Technical Sciences, University of Novi Sad.

## Features
- **Priority scheduling** – jobs with a lower priority number are processed first; jobs with equal priority keep their submission order
- **Producer–consumer model** – producer threads submit jobs while a pool of worker threads processes them
- **Bounded queue** – new jobs are rejected when the queue reaches its configured maximum size
- **Idempotency** – a job with the same ID cannot be submitted or processed twice
- **Timeout and retry** – a job that runs longer than 2 seconds is marked as failed and retried, up to 3 attempts in total, after which it is aborted
- **Asynchronous results** – each submitted job returns a handle with a `Task<int>` that completes when the job finishes
- **Event-based logging** – `JobCompleted` and `JobFailed` events write entries to a log file asynchronously
- **Periodic reports** – XML reports generated every 60 seconds with LINQ-based statistics, keeping the last 10 reports

## Architecture

```
 Producer threads ──Submit()──► Priority queue ──► Worker threads ──► JobProcessor
                                (thread-safe)                              │
                                                                           ▼
                          job_log.txt ◄── JobCompleted / JobFailed events ◄─┘
                          reports/    ◄── ReportService (every 60 s)
```

| Component | Responsibility |
|---|---|
| `ProcessingSystem` | Priority queue, worker threads, retry logic, events and logging |
| `JobProcessor` | Executes the job types |
| `ReportService` | Collects results and generates XML reports |
| `SystemConfig` | Loads settings and initial jobs from XML |

Thread synchronization uses `lock` with `Monitor.Wait` / `Monitor.PulseAll`, concurrent collections and `TaskCompletionSource` for job results.

### Job types
| Type | Description | Payload example |
|---|---|---|
| `Prime` | Counts prime numbers up to a limit in parallel; thread count is limited to 1–8 | `numbers:10_000,threads:3` |
| `IO` | Simulates reading from a device with a delay and returns a value from 0 to 100 | `delay:1_000` |

## Getting started

### Requirements
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)

### Run
```bash
git clone https://github.com/marijelaveljkovic/IndustrialProcessingSystem.git
cd IndustrialProcessingSystem
dotnet run --project IndustrialProcessingSystem
```
The system loads `SystemConfig.xml` from the current directory (a different path can be passed as the first argument), submits the initial jobs and starts the producer threads. Press **Enter** to stop.

### Output
- `job_log.txt` – one line per event:
  ```
  [2026-05-04 12:00:00.123] [COMPLETED] <JobId>, <Result>
  ```
  Statuses: `COMPLETED`, `FAILED` (attempt failed, will be retried) and `ABORT` (all attempts failed)
- `reports/report_01.xml` … `report_10.xml` – completed jobs and average execution time per job type, and aborted jobs per job type; after 10 reports the oldest file is overwritten

## Configuration
```xml
<SystemConfig>
  <WorkerCount>5</WorkerCount>
  <MaxQueueSize>100</MaxQueueSize>
  <Jobs>
    <Job Type="Prime" Payload="numbers:10_000,threads:3" Priority="1"/>
    <Job Type="IO" Payload="delay:1_000" Priority="3"/>
  </Jobs>
</SystemConfig>
```
| Setting | Description |
|---|---|
| `WorkerCount` | Number of worker threads and producer threads |
| `MaxQueueSize` | Maximum number of jobs waiting in the queue |
| `Jobs` | Initial jobs submitted at startup |

## Testing
The solution includes **43 unit tests** written with xUnit, covering job processing, scheduling and priority order, queue limits, idempotency, events, logging, reporting and configuration loading.

```bash
dotnet test
```

With code coverage:
```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
```

### Continuous integration
Every push and pull request triggers a GitHub Actions workflow that builds the solution, runs all tests and enforces a **minimum line coverage of 67%**. The coverage report is uploaded as a build artifact.

## Project structure
```
IndustrialProcessingSystem/
├── .github/workflows/ci.yml           # CI pipeline
├── IndustrialProcessingSystem/        # Application
│   ├── Config/                        # XML configuration loading
│   ├── Models/                        # Job, JobHandle, JobType
│   ├── Services/                      # ProcessingSystem, JobProcessor, ReportService
│   └── Program.cs                     # Entry point and producer threads
├── IndustrialProcessingSystem.Tests/  # xUnit tests
└── SystemConfig.xml                   # Default configuration
```

## Tech stack
C# · .NET 8 · xUnit · Coverlet · GitHub Actions