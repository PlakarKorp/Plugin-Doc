# 📦 How to Write a Plakar Plugin in Go

This guide explains **how to implement a Plakar plugin** (Exporter, Importer, or Storage) in Go, and how to use the provided **SDK** to run it.

---

## 📌 What is a Plakar Plugin?

A Plakar Plugin is an external binary that implements one of the **Plakar interfaces**:

- **Exporter**: Handles storing snapshots to disk (e.g., writing files, setting permissions)
- **Importer**: Handles scanning data to import into Plakar
- **Storage**: Handles the low-level storage backend (states, packfiles, locks)

---

## ✅ Step 1 — Implement the Interface

Each plugin **must implement the right interface** from the Plakar source tree.

Below are the definitions ⤵️

---

### `Exporter` Interface

Path: [`plakarkorp/kloset/snapshot/exporter/exporter.go`](https://github.com/PlakarKorp/kloset/blob/main/snapshot/exporter/exporter.go)

```go
type Exporter interface {
	Root() string
	CreateDirectory(pathname string) error
	StoreFile(pathname string, fp io.Reader, size int64) error
	SetPermissions(pathname string, fileinfo *objects.FileInfo) error
	Close() error
}
````

---

### `Importer` Interface

Path: [`plakarkorp/kloset/snapshot/importer/importer.go`](https://github.com/PlakarKorp/kloset/blob/main/snapshot/importer/importer.go)

```go
type Importer interface {
	Origin() string
	Type() string
	Root() string
	Scan() (<-chan *ScanResult, error)
	Close() error
}
```

---

### `Storage` Interface

Path: [`plakarkorp/kloset/storage/storage.go`](https://github.com/PlakarKorp/kloset/blob/main/storage/storage.go)

```go
type Store interface {
	Create(ctx context.Context, config []byte) error
	Open(ctx context.Context) ([]byte, error)
	Location() string
	Mode() Mode
	Size() int64

	GetStates() ([]objects.MAC, error)
	PutState(mac objects.MAC, rd io.Reader) (int64, error)
	GetState(mac objects.MAC) (io.Reader, error)
	DeleteState(mac objects.MAC) error

	GetPackfiles() ([]objects.MAC, error)
	PutPackfile(mac objects.MAC, rd io.Reader) (int64, error)
	GetPackfile(mac objects.MAC) (io.Reader, error)
	GetPackfileBlob(mac objects.MAC, offset uint64, length uint32) (io.Reader, error)
	DeletePackfile(mac objects.MAC) error

	GetLocks() ([]objects.MAC, error)
	PutLock(lockID objects.MAC, rd io.Reader) (int64, error)
	GetLock(lockID objects.MAC) (io.Reader, error)
	DeleteLock(lockID objects.MAC) error

	Close() error
}
```

---

## 🛠️ Step 2 — Write Your Implementation

Example: Implementing an **Exporter**:

```go
package myexporter

import (
	"io"
	"os"
	"path/filepath"

	"github.com/PlakarKorp/kloset/objects"
)

type LocalExporter struct {
	root string
}

func (l *LocalExporter) Root() string {
	return l.root
}

func (l *LocalExporter) CreateDirectory(pathname string) error {
	return os.MkdirAll(filepath.Join(l.root, pathname), 0755)
}

func (l *LocalExporter) StoreFile(pathname string, fp io.Reader, size int64) error {
	f, err := os.Create(filepath.Join(l.root, pathname))
	if err != nil {
		return err
	}
	defer f.Close()

	_, err = io.CopyN(f, fp, size)
	return err
}

func (l *LocalExporter) SetPermissions(pathname string, fileinfo *objects.FileInfo) error {
	return os.Chmod(filepath.Join(l.root, pathname), fileinfo.Mode())
}

func (l *LocalExporter) Close() error {
	return nil
}
```

---

## 🚀 Step 3 — Run Your Plugin

Each plugin is started via the **SDK helper**:

| Plugin Type | SDK Function      |
| ----------- | ----------------- |
| Exporter    | `sdk.RunExporter` |
| Importer    | `sdk.RunImporter` |
| Storage     | `sdk.RunStorage`  |

---

### Example `main.go` for an Exporter

```go
package main

import (
	"context"
	
	"github.com/PlakarKorp/go-kloset-sdk/sdk"
	"github.com/PlakarKorp/kloset/snapshot/exporter"
)

func NewLocalExporter(ctx context.Context, opts *exporter.Options, name string, config map[string]string) (exporter.Exporter, error) {
	return &FSExporter{
		rootDir: strings.TrimPrefix(config["location"], "local://"),
	}, nil
}

func main() {
	if err := sdk.RunExporter(NewLocalExporter); err != nil {
		panic(err)
	}
}
```

---

## ⚡ How It Works

✅ **Plakar** starts your plugin as a separate process
✅ It opens a **socket connection** to the plugin using `InitConn()`
✅ It invokes your implementation via **gRPC**
✅ The `RunExporter`, `RunImporter`, or `RunStorage` function sets up the server and serves requests

---

## ✅ Final Tips

* **Your implementation must be valid Go** and build as a binary.
* **Don’t export unnecessary types** — only export your `RunX` function.
* The **SDK** takes care of the gRPC protocol, so **you only write the business logic**.

---

## 📚 Related

* [Plakar Repo](https://github.com/PlakarKorp/plakar)
* [`snapshot/exporter`](https://github.com/PlakarKorp/kloset/tree/main/snapshot/exporter)
* [`snapshot/importer`](https://github.com/PlakarKorp/kloset/tree/main/snapshot/importer)
* [`storage`](https://github.com/PlakarKorp/kloset/tree/main/storage)

---

## 📈 Example Architecture

```plaintext
+----------------+
|     Plakar     |
+----------------+
        |
     gRPC Conn
        |
+----------------------+
|  Your Plugin Binary  |
|                      |
|  +----------------+  |
|  |  SDK Server    |  |
|  |  (RunExporter) |  |
|  +----------------+  |
|          |           |
|  +----------------+  |
|  |  Your Impl     |  |
|  |  (implements   |  |
|  |   Exporter)    |  |
+----------------------+
```

---

## 🏁 Done!

✅ That’s all you need to build your own **Plakar plugin**!
