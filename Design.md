markdown_content = """# Software Architecture Document (SAD) - Mini
## Project: Headless Extensible File Service

---

### 1. Introduction & Design Goals
The goal of this project is to build a high-performance, lightweight, and 100% headless File Service. It must operate efficiently across distributed environments, specifically tailored for a Raspberry Pi cluster running K3s, with centralized state managed in MariaDB and physical files stored on an NFS network share.

#### Core Objectives:
* **100% Headless:** Pure JSON API for browsing and searching; direct streaming for payloads.
* **Extensible Storage & Database:** Agnostic interface layers supporting switching between Local/NFS and Object Storage (S3), as well as multi-database backends (MariaDB, PostgreSQL, MongoDB).
* **Resource Efficient:** Low CPU/Memory footprint via streaming I/O (Go-based implementation) to run optimally on ARM64 single-board computers.
* **Cloud-Native & Bare-Metal Compatible:** Easily deployable as a single executable binary, a standard Docker container, or horizontally scaled inside K3s.

---

### 2. Architectural Overview
The system follows a clean, decoupled architecture utilizing the **Strategy Design Pattern** for both database persistence and physical file storage.

               ┌─────────────────────────┐
                   │    HTTP Client / UI     │
                   └────────────┬────────────┘
                                │ HTTP Requests
                                ▼
                   ┌─────────────────────────┐
                   │     Go API Router       │
                   └──────┬────────────┬─────┘
                          │            │
     ┌────────────────────┘            └────────────────────┐
     ▼                                                      ▼
┌─────────────────────────┐                            ┌─────────────────────────┐│  <> Database │                            │ <> Storage   │└────────────┬────────────┘                            └────────────┬────────────┘│                                                      │┌────────┼────────┐                                    ┌────────┼────────┐▼        ▼        ▼                                    ▼        ▼        ▼┌────────┐┌────────┐┌────────┐                          ┌────────┐┌────────┐┌────────┐│MariaDB ││Postgres││MongoDB │                          │ Local/ ││   S3   ││ Azure  ││        ││        ││        │                          │  NFS   ││ Driver ││ Blob   │└────────┘└────────┘└────────┘                          └────────┘└────────┘└────────┘
The application logic depends solely on interfaces. Concrete implementations are dynamically instantiated at startup via environment configuration.

---

### 3. Database Architecture & Abstraction

To support multi-database capability seamlessly, a standard interface maps out metadata interactions.

#### 3.1 Metadata Schema (SQL Reference)
```sql
CREATE TABLE files (
    id VARCHAR(36) PRIMARY KEY,       -- UUID v4
    name VARCHAR(255) NOT NULL,       -- Original file name
    parent_path VARCHAR(512) NOT NULL,-- Virtual folder hierarchy (e.g., "/documents/")
    size_bytes BIGINT NOT NULL,       -- Payload size
    mime_type VARCHAR(100) NOT NULL,  -- Content identifier
    storage_key VARCHAR(100) NOT NULL,-- Unique key on disk/object bucket
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_path (parent_path),
    FULLTEXT idx_search (name)
);
3.2 The Database Interface (database/db.go)Gopackage database

import "context"

type FileMetadata struct {
    ID         string
    Name       string
    ParentPath string
    SizeBytes  int64
    MimeType   string
    StorageKey string
}

type Database interface {
    Connect(dsn string) error
    SaveMetadata(ctx context.Context, meta FileMetadata) error
    GetMetadata(ctx context.Context, id string) (FileMetadata, error)
    Search(ctx context.Context, query string) ([]FileMetadata, error)
    Close() error
}
3.3 Concrete Drivers (Examples)MariaDB (database/mariadb.go)Gopackage database

import (
    "context"
    "database/sql"
    _ "[github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)"
)

type MariaDB struct {
    conn *sql.DB
}

func (m *MariaDB) Connect(dsn string) error {
    db, err := sql.Open("mysql", dsn)
    if err != nil { return err }
    m.conn = db
    return db.Ping()
}

func (m *MariaDB) SaveMetadata(ctx context.Context, meta FileMetadata) error {
    query := `INSERT INTO files (id, name, parent_path, size_bytes, mime_type, storage_key) VALUES (?, ?, ?, ?, ?, ?)`
    _, err := m.conn.ExecContext(ctx, query, meta.ID, meta.Name, meta.ParentPath, meta.SizeBytes, meta.MimeType, meta.StorageKey)
    return err
}

func (m *MariaDB) Close() error { return m.conn.Close() }
func (m *MariaDB) GetMetadata(ctx context.Context, id string) (FileMetadata, error) { return FileMetadata{}, nil }
func (m *MariaDB) Search(ctx context.Context, query string) ([]FileMetadata, error) { return nil, nil }
PostgreSQL (database/postgres.go)Gopackage database

import (
    "context"
    "database/sql"
    _ "[github.com/lib/pq](https://github.com/lib/pq)"
)

type Postgres struct {
    conn *sql.DB
}

func (p *Postgres) Connect(dsn string) error {
    db, err := sql.Open("postgres", dsn)
    if err != nil { return err }
    p.conn = db
    return db.Ping()
}

func (p *Postgres) SaveMetadata(ctx context.Context, meta FileMetadata) error {
    query := `INSERT INTO files (id, name, parent_path, size_bytes, mime_type, storage_key) VALUES ($1, $2, $3, $4, $5, $6)`
    _, err := p.conn.ExecContext(ctx, query, meta.ID, meta.Name, meta.ParentPath, meta.SizeBytes, meta.MimeType, meta.StorageKey)
    return err
}

func (p *Postgres) Close() error { return p.conn.Close() }
func (p *Postgres) GetMetadata(ctx context.Context, id string) (FileMetadata, error) { return FileMetadata{}, nil }
func (p *Postgres) Search(ctx context.Context, query string) ([]FileMetadata, error) { return nil, nil }
4. Storage Abstraction LayerFiles are treated strictly as standard input/output bytes streams (io.Reader and io.ReadCloser). This ensures that the system handles multi-gigabyte files with low, flat memory usage.4.1 Storage Interface (storage/driver.go)Gopackage storage

import "io"

type Driver interface {
    Save(storageKey string, src io.Reader) error
    Get(storageKey string) (io.ReadCloser, error)
    Delete(storageKey string) error
}
4.2 Local / NFS Driver (storage/local.go)When deployed on Kubernetes (K3s), network attached storage paths (NFS) are mapped transparently onto the file system. The application uses simple disk I/O.Gopackage storage

import (
    "io"
    "os"
    "path/filepath"
)

type LocalDriver struct {
    BasePath string
}

func (l *LocalDriver) Save(key string, src io.Reader) error {
    fullPath := filepath.Join(l.BasePath, key)
    dst, err := os.Create(fullPath)
    if err != nil { return err }
    defer dst.Close()
    
    _, err = io.Copy(dst, src) // Stream pipe allocation
    return err
}

func (l *LocalDriver) Get(key string) (io.ReadCloser, error) {
    return os.Open(filepath.Join(l.BasePath, key))
}

func (l *LocalDriver) Delete(key string) error {
    return os.Remove(filepath.Join(l.BasePath, key))
}
5. Headless REST API EndpointsAll browsing and search queries return explicit application/json payloads. Download files utilize automatic streaming pipelines.5.1 Endpoints SpecificationActionHTTP MethodEndpointDescriptionPayload / ResponseUpload FilePOST/filesStreams file payload + stores metadatamultipart/form-dataDownload FileGET/files/{id}Pipes bytes from store to network outputRaw Stream (Binary)Delete FileDELETE/files/{id}Disposes binary payload + drops DB recordJSON ConfirmationBrowse PathGET/files?path={p}Scans DB for elements matching virtual pathJSON Directory ListSearch CatalogGET/search?q={term}Wildcard fuzzy lookup across names/pathsJSON Search Results5.2 Browse Output Example (GET /files?path=/documents/)JSON{
  "current_path": "/documents/",
  "folders": ["finance", "engineering"],
  "files": [
    {
      "id": "e4bca23d-71b3-4f9e-a128-d8902d13bb22",
      "name": "architecture_blueprint.pdf",
      "size_bytes": 10485760,
      "mime_type": "application/pdf",
      "created_at": "2026-06-25T13:12:00Z"
    }
  ]
}
6. Cluster Deployment Configuration (K3s & Docker)Since the service instances are completely stateless, multiple replicas can be run concurrently across the Raspberry Pi nodes. Traffic can be balanced dynamically using an Ingress layer.6.1 Kubernetes Deployment & PVC Manifest (deployment.yaml)YAMLapiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fileservice-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany # Allows concurrent read/write across multiple Pi cluster nodes
  resources:
    requests:
      storage: 500Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: headless-file-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: file-service
  template:
    metadata:
      labels:
        app: file-service
    spec:
      containers:
      - name: file-service
        image: custom-registry/headless-file-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_TYPE
          value: "mariadb"
        - name: DB_DSN
          value: "user:password@tcp(mariadb-cluster-ip:3306)/fileservice"
        - name: STORAGE_PATH
          value: "/mnt/nfs-storage"
        volumeMounts:
        - name: nfs-volume
          mountPath: "/mnt/nfs-storage"
      volumes:
      - name: nfs-volume
        persistentVolumeClaim:
          claimName: fileservice-nfs-pvc
"""filename = "headless_file_service_sad.md"with open(filename, "w") as f:f.write(markdown_content)print(filename)Your Markdown (.md) file containing the mini Software Architecture Document (SAD) is ready:
[file-tag: code-generated-file-0-1782385960205329510]

---

### Mini Software Architecture Document (SAD) Preview

```markdown
# Software Architecture Document (SAD) - Mini
## Project: Headless Extensible File Service

### 1. Introduction & Design Goals
The goal of this project is to build a high-performance, lightweight, and 100% headless File Service. It operates efficiently across distributed environments, specifically tailored for a Raspberry Pi cluster running K3s, with centralized state managed in MariaDB and physical files stored on an NFS network share.

### 2. Architectural Overview
The system follows a clean, decoupled architecture utilizing the **Strategy Design Pattern** for both database persistence and physical file storage. 

Application logic depends solely on interfaces. Concrete implementations (e.g., `Postgres`, `MariaDB`, `MongoDB`) are dynamically instantiated at startup via environment configuration (`DB_TYPE`).

### 3. Database Architecture & Abstraction
To support multi-database capability seamlessly, a standard interface maps out metadata interactions.

#### The Database Interface (`database/db.go`)
```go
type Database interface {
    Connect(dsn string) error
    SaveMetadata(ctx context.Context, meta FileMetadata) error
    GetMetadata(ctx context.Context, id string) (FileMetadata, error)
    Search(ctx context.Context, query string) ([]FileMetadata, error)
    Close() error
}
4. Storage Abstraction LayerFiles are treated strictly as standard input/output byte streams (io.Reader). This ensures that the system handles multi-gigabyte files with low, flat memory usage across the cluster.Storage Interface (storage/driver.go)Gotype Driver interface {
    Save(storageKey string, src io.Reader) error
    Get(storageKey string) (io.ReadCloser, error)
    Delete(storageKey string) error
}
5. Headless REST API EndpointsAll browsing and search queries return explicit application/json payloads.GET /files?path=/documents/ -> Returns file metadata and sub-folder strings.GET /search?q=query -> Wildcard lookup across virtual names/paths.6. Cluster Deployment Configuration (K3s & Docker)Since the service instances are completely stateless, multiple replicas run concurrently across the Raspberry Pi nodes using K3s ReadWriteMany Persistent Volume Claims to share the NFS mount.
