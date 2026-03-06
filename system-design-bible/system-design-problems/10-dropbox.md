# System Design: Dropbox

## Problem

Design a file sync and storage system: upload files, store them in the cloud, and keep them in sync across multiple devices. Support large files, deduplication, and efficient sync (only changed parts).

## Requirements

**Functional:** Upload/download files; sync across devices (detect changes, transfer deltas); folder structure; sharing (optional); version history (optional).

**Non-functional:** Efficient use of bandwidth (chunking, dedup); handle large files; consistency across devices; durability.

## High-Level Design

```
  Client (desktop/mobile) → API → Metadata service (paths, versions, blocks)
                                        ↓
  Block store (chunked content, dedup by hash) ← Client uploads blocks
  Client: local index (file → block hashes); sync by comparing with server index
```

- **Upload:** Client chunks file (e.g. 4 MB); hash each chunk; send hashes to server; server returns “which hashes are new”; client uploads only new blocks; server stores blocks (content-addressable); metadata: file path → list of block hashes.
- **Download/Sync:** Client asks for metadata (path → block hashes); compares with local; downloads missing blocks; assembles file.
- **Change detection:** Client watches filesystem (or polls); recompute chunk hashes; only changed chunks trigger upload.

## Key Components

- **Metadata service:** Files and folders: path, block_hashes, mtime, version. Per-user namespace; list and get metadata. Shard by user_id.
- **Block store:** Content-addressable; key = hash(block content); value = block. Dedup: same content → one block. Object store (S3) or custom.
- **Client:** Chunking (fixed size or rolling hash for better dedup); hash; sync protocol (list metadata, diff, upload/download blocks); local DB of synced state.
- **Sync protocol:** Efficient: only transfer missing blocks; batch metadata updates; conflict resolution (e.g. last-write-wins or keep both with rename).

## Data Model

- **files_metadata:** user_id, path, block_hashes[], size, mtime, version.
- **blocks:** block_hash (key), content (value). Stored in object store.
- **users:** user_id, quota, etc.

## Scale

- Millions of users; billions of files. Metadata: DB sharded by user; cache hot metadata. Block store: object store scales; dedup reduces storage. Bandwidth: chunking and dedup reduce transfer; CDN for download if needed.

## Trade-offs

- **Chunk size:** Smaller = better dedup, more metadata and overhead. Larger = less overhead, worse dedup. Common: 4 MB or rolling hash (content-defined).
- **Conflict:** Last-write-wins vs keep-both (conflict copy); or operational merge for simple cases.
- **Block store:** Content-addressable enables dedup; need to handle block GC if no file references it (reference count or background scan).

## Failure & Operations

- **Consistency:** Metadata and block store eventually consistent; client retries and reconciles on conflict.
- **Durability:** Blocks in object store with replication; metadata in DB with backups.
- **Monitoring:** Sync latency, upload/download throughput, storage and dedup ratio, conflict rate.
