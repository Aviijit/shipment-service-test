# Shipment Service

A production-grade Spring Boot service that accepts large CSV files and imports them as shipments and shipment events asynchronously — designed to handle up to 50,000 rows without memory abuse or HTTP timeouts.

---

## Tech Stack

| Concern | Choice |
|---|---|
| Language | Kotlin |
| Framework | Spring Boot 4.0.6 |
| Database | PostgreSQL 15 |
| ORM | Spring Data JPA / Hibernate |
| Schema Migration | Flyway |
| CSV Parsing | Apache Commons CSV |
| Build Tool | Gradle (Kotlin DSL) |
| Testing | JUnit 5 + MockK |

---

## Architecture Overview

```
Client
  │
  │  POST /api/imports (multipart CSV)
  ▼
ImportController
  │  Validates file, enforces 50MB limit (returns 413 if exceeded)
  ▼
ImportOrchestrator
  │  Saves file to disk, persists Import record with status PENDING
  ▼
CsvImportJob  (@Async)
  │  Returns HTTP 201 immediately — processing continues in background
  │
  ├──▶ CsvRowParser
  │      Streams CSV row-by-row via BufferedReader
  │      Never loads entire file into memory
  │      Groups rows by shipment_id in a single pass
  │
  └──▶ ShipmentUpsertService  (@Transactional per shipment)
         Acquires pessimistic write lock on shipment row
         Creates shipment if not found, updates if exists
         Deletes and re-inserts all events atomically
         Deduplicates event_type within same shipment (last row wins)
```

---

## Design Decisions

### Async Processing
Upload returns `HTTP 201 Created` immediately with an `importId`. A background thread pool processes the file asynchronously. The client polls `GET /api/imports/{id}` to track progress. This approach avoids HTTP timeouts on large files and gives the system room to handle concurrent uploads gracefully.

### Streaming CSV Parse
Apache Commons CSV reads the file row-by-row via `BufferedReader` —— the entire file is never loaded into memory at once. Rows are grouped by `shipment_id` in a single streaming pass, keeping RAM usage flat regardless of file size.

### Transactional Integrity Per Shipment
Each shipment is upserted in its own `@Transactional` boundary. A failure in one shipment does not affect others — their data remains intact. Events are deleted and re-inserted atomically within the same transaction, ensuring no partial writes.

### Multi-Writer Safety
`SELECT FOR UPDATE` (pessimistic write lock) is acquired on the shipment row before any mutation. Two concurrent imports targeting the same shipment will serialize rather than corrupt each other's data.

### Event Overwrite Strategy
Each import completely replaces all events for a shipment. This satisfies the requirement of one event per type per shipment and is enforced at both the application level (deduplication) and the database level (unique constraint on `shipment_id, event_type`).

### Error Isolation
Invalid rows (bad UUID, unknown event_type, blank required fields) are skipped and counted as `failed_rows` — they do not abort the entire import. Shipment-level failures are also isolated, leaving all other shipments unaffected.

---


## Prerequisites
 
| Option | Requirements |
|---|---|
| Docker (recommended) | Docker + Docker Compose |
| Local development | Java 21+, Docker (for PostgreSQL) |
 
---
 
## Quickstart — Docker (Recommended)
 
The fastest way to run the service. No Java or Gradle installation required.
 
### Step 1 — Clone the repository
 
```bash
git clone https://github.com/Aviijit/shipment-service-test.git
cd shippio-csv-import
```
 
### Step 2 — Start everything
 
```bash
docker-compose up --build
```
 
This will:
- Pull PostgreSQL 15 image
- Build the Spring Boot application image
- Run Flyway migrations automatically
- Load sample shipment data into the database
- Start the service on port `8080`
### Step 3 — Verify the service is running
 
```bash
curl http://localhost:8080/actuator/health
```
 
Expected response:
```json
{ "status": "UP" }
```
 
### Step 4 — Open Swagger UI
 
```
http://localhost:8080
```
 
The application redirects to Swagger UI automatically. Sample shipment data is pre-loaded so you can test the API immediately without uploading a CSV first.
 
### Step 5 — Try the API
 
```bash
# List pre-loaded shipments
curl http://localhost:8080/api/shipments | jq .
 
# Upload a CSV file
curl -X POST http://localhost:8080/api/imports \
  -F "file=@/path/to/your/file.csv"
 
# Poll import status (replace with returned id)
curl http://localhost:8080/api/imports/{id} | jq .
```
 
### Stopping the service
 
```bash
docker-compose down          # stop containers
docker-compose down -v       # stop containers and remove volumes (clean slate)
```
 
---

## API Documentation
 
Interactive API documentation is available via Swagger UI once the application is running:
 
```
http://localhost:8080/swagger-ui.html
```
 
The Swagger UI allows you to explore all endpoints, view request/response schemas, and execute API calls directly from the browser — no additional tooling required.
 
---
 
## API Endpoints
 
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/imports` | Upload CSV and trigger async import |
| GET | `/api/imports` | List all imports (newest first) |
| GET | `/api/imports/{id}` | Poll import status and progress |
| GET | `/api/shipments` | List shipments (paginated) |
| GET | `/api/shipments/{id}` | Get single shipment with all events |
| GET | `/actuator/health` | Health check |
| GET | `/swagger-ui.html` | Interactive API documentation |
 
---
 
## CSV Format
 
```csv
shipment_id,customer_team_ref,event_type,status,event_completed_at
9389fd55-b451-49f5-aac4-6add777caca3,team-9,departure,completed,2025-09-20T08:00:00Z
9389fd55-b451-49f5-aac4-6add777caca3,team-9,arrival,in_progress,2025-09-25T14:20:00Z
9389fd55-b451-49f5-aac4-6add777caca3,team-9,delivery,pending,
12adbeef-22ab-4d9f-9f0e-6add777cbb11,team-7,departure,completed,2025-09-19T09:15:00Z
```
 
**Supported `event_type` values:** `pickup`, `packing`, `departure`, `transship`, `arrival`, `delivery`
 
**Supported `status` values:** `pending`, `in_progress`, `completed`, `failed`
 
`event_completed_at` is optional — leave blank for `pending` or `in_progress` events.
 
---
 
 
## Configuration
 
Key settings in `application.yaml`:
 
| Property | Default | Description |
|---|---|---|
| `app.file-storage.upload-dir` | `./uploads` | Local directory for uploaded CSV files |
| `spring.servlet.multipart.max-file-size` | `50MB` | Max upload size — returns 413 if exceeded |
| `app.async.core-pool-size` | `2` | Background import thread pool core size |
| `app.async.max-pool-size` | `5` | Background import thread pool max size |
| `app.async.queue-capacity` | `100` | Import job queue capacity |
| `server.tomcat.threads.max` | `100` | Max HTTP threads |
| `spring.datasource.hikari.maximum-pool-size` | `10` | Max DB connections |
 
---
 