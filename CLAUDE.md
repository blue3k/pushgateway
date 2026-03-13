# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
go build ./...
make build          # uses promu to produce a versioned binary

# Test
go test ./...
go test ./storage/... -run TestFoo   # run a single test

# Format / lint
go fmt ./...
make style          # gofmt + import ordering check
make lint           # golangci-lint (Linux/macOS only)
make unused         # check for unused code

# Regenerate embedded assets (after changing resources/ or handler/templates)
make assets
```

## Architecture

The pushgateway is a single-binary HTTP server that buffers Prometheus metrics pushed by jobs and re-exposes them for scraping.

### Request flow

```
HTTP PUT/POST /metrics/job/:job[/*labels]
  → handler.Push()          (handler/push.go)
  → ms.SubmitWriteRequest() → writeQueue chan WriteRequest
  → DiskMetricStore.loop()  (storage/diskmetricstore.go)
      → processWriteRequest()  writes to metricGroups map
      → persist()              gob-encodes map to disk (throttled)

HTTP GET /metrics
  → prometheus.Gatherers{DefaultGatherer, ms.GetMetricFamilies()}
```

DELETE requests set `MetricFamilies = nil` in the WriteRequest, which causes `processWriteRequest` to delete the group.

### Storage layer (`storage/`)

- **`interface.go`** — public types: `MetricStore`, `WriteRequest`, `MetricGroup`, `GroupingKeyToMetricGroup`, `TimestampedMetricFamily`
- **`diskmetricstore.go`** — sole implementation. All writes are serialized through a single `writeQueue` channel processed by one `loop()` goroutine. `metricGroups` (the main map) is protected by `sync.RWMutex`; reads (`GetMetricFamilies`, `GetMetricFamiliesMap`) take a read lock, `processWriteRequest` takes a write lock.

**Grouping key**: metrics are grouped by a string key formed by sorting all label name/value pairs and joining them with `model.SeparatorByte` — see `groupingKeyFor()`.

**TTL expiry** (this fork's addition): `DiskMetricStore` maintains a `*list.List` (`ttlList`) and a `map[string]*list.Element` (`ttlIndex`) — both accessed exclusively from `loop()` so they need no separate lock. Each push moves the group's element to the back (most-recently-pushed = back, soonest-to-expire = front). A `time.AfterFunc` timer fires exactly when the front element expires; `expireFromList()` then pops from the front until it finds a non-expired entry. The global TTL is set via `--metric-ttl` (default `0` = disabled).

**Persistence**: on writes, `persist()` gob-encodes the entire `metricGroups` map to a temp file then atomically renames it. On startup, `restore()` decodes the file and rebuilds the `ttlList`/`ttlIndex` from loaded groups sorted by `ExpiresAt`.

**Consistency check**: before applying a write, `checkWriteRequest()` may run an expensive check by cloning the store into a temporary `DiskMetricStore` and running `prometheus.Gatherers.Gather()` to detect inconsistent metric families. This check only runs when the `WriteRequest.Done` channel is non-nil (i.e. synchronous pushes when `--push.disable-consistency-check` is not set).

### Handler layer (`handler/`)

- `push.go` — `Push()` parses the URL path for job+labels, parses the metric body (protobuf delimited or text), and submits a `WriteRequest`. PUT → `Replace: true`, POST → `Replace: false`.
- `delete.go` — submits a `WriteRequest` with `MetricFamilies: nil`.
- `wipe.go` — admin endpoint; iterates all groups and deletes each.
- `status.go` — HTML status page, calls `ms.GetMetricFamiliesMap()`.

### API layer (`api/v1/`)

JSON REST API mounted at `/api/v1/`. Reads are served from `ms.GetMetricFamiliesMap()`. The wipe endpoint (`PUT /api/v1/admin/wipe`) is mounted separately in `main.go` only when `--web.enable-admin-api` is set.

### Key flag interactions

| Flag | Effect |
|---|---|
| `--persistence.file` | enables disk persistence; empty = memory only |
| `--metric-ttl` | global TTL for all metric groups (0 = never expire) |
| `--push.disable-consistency-check` | skips the expensive clone+gather check on every push |
| `--web.enable-admin-api` | enables `PUT /api/v1/admin/wipe` |
| `--push.enable-utf8-names` | switches label validation to UTF-8 mode |
