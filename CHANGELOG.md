# Changelog

All notable changes to the GFS distributed file system.

## [Unreleased]

### Added

- **Orphan chunk cleanup at storage server startup.** When a storage server
  starts, it queries the naming server for the list of chunk IDs it *should*
  hold (via the new `ListExpectedChunks` RPC) and deletes any chunks on disk
  that are no longer in metadata. This reclaims disk space wasted by replica
  migrations during self-healing — when a failed server returns, the chunks
  that were moved to other servers are no longer needed locally.
  - New RPC: `NamingServer.ListExpectedChunks(address) → chunk_ids`
  - New metadata query: `MetadataStore.list_chunk_ids_for(address)`

- **Garbage collection for interrupted writes.** Pending files whose write was
  interrupted (client crash, timeout, etc.) are now automatically cleaned up.
  A background thread in the naming server deletes pending files older than 60
  seconds (configurable via `cleanup_max_age`). Orphaned chunk data on storage
  servers is deleted on a best-effort basis before removing the metadata.
  - Added `created_at` timestamp column to the `files` table with automatic
    migration for existing databases.
  - `MetadataStore.list_stale_pending(max_age_seconds)` returns stale pending
    files with full chunk location info.

### Fixed

- **Performance: synchronous metrics collection degraded write throughput.**  
  The `_refresh_cluster_metrics()` method in the naming server performed a full
  scan of all committed chunks (including an N+1 SQL query for replicas) on
  **every** RPC call (CreateFile, CommitFile, Heartbeat, GetFile, etc.). The
  same pattern existed in the storage server where `refresh_metrics()` did
  `os.listdir()` + `os.path.getsize()` for every chunk on every `StoreChunk`
  and `DeleteChunk`. As the number of chunks grew, each individual operation
  became linearly slower.

  - Naming server: `_refresh_cluster_metrics()` now runs on a background thread
    every 10 seconds instead of synchronously on every RPC.
  - Storage server: `refresh_metrics()` removed from `StoreChunk` and
    `DeleteChunk` hot paths; it continues to run periodically in the heartbeat
    loop (every 5 seconds). Byte counters remain incremental.
  - `MetadataStore.list_committed_chunks()`: replaced N+1 per-chunk `SELECT`
    queries with a single `LEFT JOIN replicas` query.

- **`CreateFile` timeout and message size for large files.**  
  - `create_pending()` uses `executemany` with generator expressions instead
    of per-row `execute()` calls (4M rows → single batch per table).
  - SQLite: WAL journal mode and `synchronous=NORMAL` for better concurrent
    read/write throughput during large transactions.
  - gRPC message size limit raised from 4 MB to 256 MB (a 1 GB file produces
    a ~90 MB `CreateFileResponse` with ~977K `ChunkPlacement` entries).
  - CLI: `--timeout` flag (default 10s) to extend gRPC deadline for large files.

## [0.1.0] — 2026-06-22

### Added

- GFS-inspired distributed file system with 1 KB fixed-size chunks.
- Naming server (metadata authority) with SQLite-backed metadata persistence.
- Storage server (chunkserver) with atomic chunk writes via temp-file rename.
- Client library with file-level read/write/delete/size/list operations.
- Configurable replication factor (default 3) with round-robin placement.
- Self-healing: periodic scan of committed chunks to repair under-replicated
  replicas after storage server failures.
- Docker Compose cluster (1 naming + 4 storage servers).
- Prometheus metrics and Grafana monitoring dashboards.
