# Upload Video Flow

## Overview

This document describes the complete lifecycle of a video upload request, from the moment a user uploads a video until the processing is completed and the final result becomes available.

---

## Sequence Flow

```
Client
 |
 | Upload Video
 v
Laravel API
 |
 | Validate Request
 | Save Original File
 | Save Video Metadata (MySQL)
 |
 | gRPC Process Request
 v
Go Processor Service
 |
 | Create Processing Job
 | Publish Job
 v
RabbitMQ Queue
 |
 | Consume Job
 v
Worker Pool
 |
 | Execute FFmpeg Processing
 |
 | Generate:
 | - 360p Video
 | - 720p Video
 | - 1080p Video
 | - Thumbnail Image
 |
 | Update Processing Status
 v
Laravel API / Database
 |
 | Return Video Status
 v
Client
```

---

## Step-by-Step Explanation

### 1. Video Upload

The client sends an HTTP request to the Laravel API with the video file.

Responsibilities:

* Validate file type and size
* Authenticate the user
* Store the original video file
* Create a video record in the database with `PENDING` status

---

### 2. Processing Request

After storing the video metadata, Laravel sends a gRPC request to the Go Processor Service.

Request example:

```json
{
    "video_id": 101,
    "path": "storage/videos/original/abc123.mp4"
}
```

The purpose of this request is to start the asynchronous processing pipeline.

---

### 3. Job Creation

The Go Processor receives the request and creates a processing job.

The job is published to RabbitMQ so that it can be processed asynchronously by workers.

---

### 4. Worker Processing

Workers continuously consume jobs from RabbitMQ.

Each worker performs:

* Download/read original video
* Execute FFmpeg commands
* Generate multiple video qualities
* Generate thumbnail image
* Handle errors and retries

---

### 5. Processing Completion

After successful processing:

* Video files are stored in their destination location
* The video status changes to `COMPLETED`
* Generated qualities and thumbnail information are saved

If processing fails:

* Status changes to `FAILED`
* Error information is recorded
* Retry mechanism may be triggered

---

## Final Video States

The video can have the following states:

| Status     | Description                               |
| ---------- | ----------------------------------------- |
| PENDING    | Video uploaded and waiting for processing |
| PROCESSING | Video is currently being processed        |
| COMPLETED  | Video processing finished successfully    |
| FAILED     | Video processing failed                   |

---

## Main Components

### Laravel API

Responsible for:

* Authentication & Authorization
* Video Upload
* Metadata Management
* Database Management
* Sending gRPC requests to Processor Service

---

### Go Processor Service

Responsible for:

* Receiving gRPC requests
* Creating processing jobs
* Publishing messages to RabbitMQ
* Managing workers
* Executing video processing logic

---

### RabbitMQ

Responsible for:

* Decoupling services
* Buffering processing jobs
* Enabling asynchronous communication

---

### Worker Pool

Responsible for:

* Concurrent job processing
* Resource management
* Retry and failure handling

---

## Architecture Goal

The main goal of this architecture is to build a scalable and asynchronous video processing platform where heavy tasks are separated from the main API, allowing the system to handle thousands of video processing requests efficiently.
