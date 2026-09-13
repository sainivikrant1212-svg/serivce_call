# Local Filesystem API

A secure, sandboxed REST API for browsing and managing files on the host machine. Built with Spring Boot 4 and Java 17.

All file operations are restricted to a configured root directory. Path traversal is blocked at the I/O layer before any read, write, or delete occurs.

## Features

- REST endpoints for list, metadata, upload, download (streaming), search, and delete
- API key and optional JWT authentication
- Apache Tika MIME-type detection for documents, images, videos, JSON, CSV, and more
- Path sandboxing with canonical root resolution
- Server-Sent Events (SSE) watcher for real-time file change notifications
- Layered architecture: controllers → services → filesystem I/O

## Project layout

```
src/main/java/com/example/filesystem/
├── config/          # Security, properties, startup initialization
├── controller/      # REST routing
├── dto/             # Request/response models
├── exception/       # Error handling
├── io/              # Path validation, MIME detection, filesystem I/O
├── security/        # API key + JWT filters
├── service/         # Business logic
└── watcher/         # SSE file change notifications
```

## Prerequisites

- Java 17+
- Maven 3.9+ (or use the included `./mvnw` wrapper)

## Quick start

```bash
./mvnw spring-boot:run
```

That's it. The API starts on `http://localhost:8080`, serves files from `/Users/vikrantsaini1.vc`, and uses API key `dev-api-key`.

```bash
curl -H "X-API-Key: dev-api-key" http://localhost:8080/api/v1/files
```

To change the root folder or API key, edit `src/main/resources/application.yml`:

```yaml
filesystem:
  root-path: /Users/vikrantsaini1.vc

security:
  api-key: dev-api-key     # sent in X-API-Key header
```

## Authentication

### API key (default)

```bash
curl -H "X-API-Key: dev-api-key" http://localhost:8080/api/v1/files
```

### JWT (optional)

Enable in `application.yml` (`security.jwt.enabled: true`) and set `security.jwt.secret`.

```bash
# Issue a token (no auth required on this endpoint when JWT is enabled)
curl -X POST http://localhost:8080/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"clientId":"my-app"}'

# Use the token
curl -H "Authorization: Bearer <token>" http://localhost:8080/api/v1/files
```

## API reference

All paths below are relative to the configured root. Use `/` for the root directory.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/health` | Health check (no auth) |
| `GET` | `/api/v1/files?path=/` | List directory contents |
| `GET` | `/api/v1/files/metadata?path=/docs/report.pdf` | File or directory metadata |
| `GET` | `/api/v1/files/download?path=/docs/report.pdf` | Stream file download |
| `POST` | `/api/v1/files/upload?path=/docs` | Upload file (`multipart/form-data`, field `file`) |
| `DELETE` | `/api/v1/files?path=/docs/report.pdf` | Delete file or directory tree |
| `GET` | `/api/v1/files/search?q=report&path=/` | Search by filename |
| `GET` | `/api/v1/files/watch?path=/docs` | SSE stream of file changes |
| `POST` | `/api/v1/auth/token` | Issue JWT (when enabled) |

### Example requests

```bash
# List root directory
curl -s -H "X-API-Key: dev-api-key" "http://localhost:8080/api/v1/files?path=/"

# Upload a file
curl -s -H "X-API-Key: dev-api-key" \
  -F "file=@./report.pdf" \
  "http://localhost:8080/api/v1/files/upload?path=/documents"

# Get metadata
curl -s -H "X-API-Key: dev-api-key" \
  "http://localhost:8080/api/v1/files/metadata?path=/documents/report.pdf"

# Download
curl -H "X-API-Key: dev-api-key" \
  "http://localhost:8080/api/v1/files/download?path=/documents/report.pdf" \
  -o report.pdf

# Search
curl -s -H "X-API-Key: dev-api-key" \
  "http://localhost:8080/api/v1/files/search?q=report&path=/"

# Delete
curl -s -X DELETE -H "X-API-Key: dev-api-key" \
  "http://localhost:8080/api/v1/files?path=/documents/report.pdf"

# Watch for changes (SSE)
curl -N -H "X-API-Key: dev-api-key" \
  "http://localhost:8080/api/v1/files/watch?path=/documents"
```

### Response shapes

**Directory listing**

```json
{
  "path": "/documents",
  "entries": [
    {
      "name": "report.pdf",
      "path": "/documents/report.pdf",
      "directory": false,
      "size": 204800,
      "mimeType": "application/pdf",
      "createdAt": "2026-09-13T08:00:00Z",
      "modifiedAt": "2026-09-13T08:05:00Z"
    }
  ]
}
```

**Error**

```json
{
  "timestamp": "2026-09-13T08:00:00Z",
  "status": 403,
  "error": "PATH_ACCESS_DENIED",
  "message": "Path escapes configured root directory"
}
```

## Security considerations

1. **Sandboxing** — Only paths under `filesystem.root-path` are accessible. `../` and absolute paths outside the root are rejected.
2. **Authentication** — Change `security.api-key` in `application.yml` before deployment. Use HTTPS in production.
3. **Least privilege** — Run the service with OS-level permissions limited to the root directory it needs.
4. **Upload limits** — Default max upload is 100MB (`spring.servlet.multipart.max-file-size` in `application.yml`).
5. **No anonymous access** — All file endpoints require a valid API key or JWT except `/api/v1/health` and `/api/v1/auth/token`.
6. **Logging** — File paths may appear in logs; avoid logging sensitive filenames in regulated environments.

## Development

```bash
# Run tests
./mvnw test

# Build JAR
./mvnw package

# Run packaged JAR
java -jar target/service-1-0.0.1-SNAPSHOT.jar
```

## SSE watcher events

The watch endpoint emits:

- `connected` — subscription established
- `file-change` — `ENTRY_CREATE`, `ENTRY_MODIFY`, or `ENTRY_DELETE`

Example event payload:

```json
{
  "eventType": "ENTRY_MODIFY",
  "path": "/documents/report.pdf",
  "name": "report.pdf",
  "timestamp": "2026-09-13T08:10:00Z"
}
```
