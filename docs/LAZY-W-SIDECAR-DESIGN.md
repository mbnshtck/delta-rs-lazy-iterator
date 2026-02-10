# Lazy Iterator with Sidecar: Delta Lake Metadata Optimization for ADX/Kusto

## Memory-Efficient Streaming with Ingestion-Side Metadata Optimization

**Version:** 2.0  
**Date:** February 2026  
**Status:** Design Document  
**Target Client:** Azure Data Explorer (ADX / Kusto) — external table queries  
**Storage Backend:** Azure Blob Storage (ADLS Gen2)  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Solution Architecture: Project Sidecar](#3-solution-architecture-project-sidecar)
4. [Component 1: Ingestion Pipeline — Sidecar Generation](#4-component-1-ingestion-pipeline--sidecar-generation)
5. [Component 2: delta-rs Enhancement — Streaming Metadata Provider](#5-component-2-delta-rs-enhancement--streaming-metadata-provider)
6. [Component 3: Kusto Integration](#6-component-3-kusto-integration)
7. [The Query Pipeline](#7-the-query-pipeline)
8. [Action Frequency Analysis (ADX Context)](#8-action-frequency-analysis-adx-context)
9. [Hourly Virtual Partitioning](#9-hourly-virtual-partitioning)
10. [Memory Analysis](#10-memory-analysis)
11. [Correctness & Safety](#11-correctness--safety)
12. [API Design](#12-api-design)
- [Appendix A: Enhancement Ladder](#appendix-a-enhancement-ladder--performance--memory-at-each-level)
- [Appendix B: Implementation Phases](#appendix-b-implementation-phases)
- [Appendix C: Beyond L7 — Further Optimizations & Edge Cases](#appendix-c-beyond-l7--further-optimizations--edge-cases)

---

## 1. Executive Summary

This document describes **Project Sidecar** — a two-pronged approach to solving OOM failures and sequential blocking when querying huge Delta Lake V1 tables from Azure Data Explorer (Kusto).

### The Two Prongs

| Prong | Where | What |
|-------|-------|------|
| **Ingestion-side** | Your pipeline (you own it) | Write a "V2-lite" sidecar Parquet alongside the V1 checkpoint |
| **Reader-side** | delta-rs (contribution to Kusto) | Stream the sidecar with parallel log tailing, not the monolithic checkpoint |

### Why Both Are Needed

- **Sidecar alone** doesn't help if the reader still loads everything into memory
- **Lazy iterator alone** still suffers from V1's JSON-stats parsing overhead
- **Together**: the ingestion pipeline pre-computes optimized metadata, and the reader streams it with bounded memory

### Key Insight from ADX Context

Your tables are **append-heavy** (99%+ `add` actions). Removes are rare (only from OPTIMIZE). This means:

- The tombstone set is almost always **empty or tiny**
- The sidecar is the **primary source of truth** for query planning
- Log tailing (JSON commits after checkpoint) is **fast and cheap**
- **Simple forward-only algorithm** — fetch commits in parallel, stream sidecar with row-group pruning
- **All valid Delta features supported** — schema evolution, protocol upgrades, deletion vectors, etc. are handled transparently

### Result

| Metric | Current V1 | With Sidecar |
|--------|-----------|--------------|
| Memory for 10M files | 10+ GB | ~50 MB |
| Time to first file | Minutes | < 200 ms |
| Stats parsing | JSON deserialization (slow) | Native Parquet columns (instant) |
| Query parallelism | Blocked until full load | Pipelined immediately |
| Kusto compatibility | ✅ V1 preserved | ✅ V1 preserved + sidecar bonus |

---

## 2. Problem Statement

### Context

- **Azure Data Explorer (Kusto)** queries these tables as **external tables**
- Kusto uses **delta-rs** internally as its Delta Lake driver
- Tables are V1 format (for compatibility), no partitioning currently used
- Some tables have **millions of files**, causing OOM

### The Three Problems

#### Problem 1: OOM from V1 Checkpoint Loading

In V1, statistics are stored as **JSON strings** inside the checkpoint Parquet:

```json
{"numRecords":1000,"minValues":{"id":1,"ts":"2026-01-01"},"maxValues":{"id":999,"ts":"2026-01-31"}}
```

When delta-rs reads this:
1. Read Parquet row → extract JSON string (~1-10 KB per file)
2. Parse JSON string into Rust objects → **3-10x memory expansion**
3. Hold all objects in memory → **OOM for millions of files**

For a table with 10M files and 1000-column schema, the stats JSON alone can be 50+ GB in memory.

#### Problem 2: Sequential Blocking

```
Current:  [====== Load Full Checkpoint ======][== Query ==]
                      ▲                            ▲
                 CPU/GPU idle                 Finally starts
                 Network busy                   working
```

The Kusto query engine is **highly parallel** but sits idle while delta-rs builds the full file list.

#### Problem 3: No Partition Pruning

Without partitions, delta-rs cannot skip irrelevant metadata. A query for `WHERE timestamp > ago(1h)` must still load metadata for **all** files across all time ranges.

---

## 3. Solution Architecture: Project Sidecar

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INGESTION PIPELINE                               │
│                                                                     │
│  On Checkpoint Trigger (every N commits):                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  1. Calculate table state (standard log replay)                │ │
│  │  2. Write V1 checkpoint  → _delta_log/0000N.checkpoint.parquet │ │
│  │  3. Write V2-lite sidecar → _delta_log/_sidecars/vN.parquet    │ │
│  │     • Stats as NATIVE Parquet columns (not JSON)               │ │
│  │     • Sorted by partition column (hour) for row-group pruning  │ │
│  │  4. Write manifest → _delta_log/_sidecars/vN.manifest.json     │ │
│  │     • Pre-extracted row group index for zero-latency discovery │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AZURE BLOB STORAGE                               │
│                                                                     │
│  table_root/                                                        │
│  ├── _delta_log/                                                    │
│  │   ├── _last_checkpoint                                           │
│  │   ├── 00001000.checkpoint.parquet    ← V1 (for legacy readers)   │
│  │   ├── 00001001.json                                              │
│  │   ├── 00001002.json                                              │
│  │   ├── 00001010.json                  ← Current version           │
│  │   └── _sidecars/                                                 │
│  │       ├── v1000.parquet              ← Optimized metadata        │
│  │       └── v1000.manifest.json        ← Row group index (~5 KB)   │
│  ├── data/                                                          │
│  │   ├── part-00001.parquet                                         │
│  │   └── ...                                                        │
│  └── (optional) _event_hour=2026020110/                             │
│      └── part-00001.parquet             ← Hour-aligned files        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│            DELTA-RS (Enhanced)                                      │
│                                                                     │
│  Query Pipeline:                                                    │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  1. Read _last_checkpoint → version C                          │ │
│  │  2. PARALLEL: Fetch JSONs C+1..V  +  GET manifest vC.json      │ │
│  │  3. Build tombstone set from removes in JSONs                  │ │
│  │  4. Yield add actions from JSONs immediately                   │ │
│  │  5. Stream sidecar row-groups with tombstone filter            │ │
│  │     • Column projection: only read needed stat columns         │ │
│  │     • Row-group pruning: skip non-matching partitions          │ │
│  │  6. Early termination on LIMIT                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Fallback: If no sidecar → use V1 checkpoint (standard path)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    KUSTO QUERY ENGINE                               │
│                                                                     │
│  • Receives file paths as a STREAM (not a full list)                │
│  • Starts parallel Parquet reads immediately                        │
│  • Early termination for `| take 100` queries                       │
│  • Memory bounded: never holds full file list                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **V1 compatibility preserved** — Legacy readers (Spark, Trino, older Kusto) use the standard V1 checkpoint unchanged
2. **Sidecar is additive** — It's a "bonus" file; if missing, the reader falls back gracefully
3. **Ingestion owns optimization** — The cost of building the sidecar is paid once at write time, not on every query
4. **Reader streams, never loads** — delta-rs never materializes the full file list
5. **Sorted metadata guarantees pruning** — Row-group skipping is deterministic, not probabilistic

---

## 4. Component 1: Ingestion Pipeline — Sidecar Generation

### 4.1 Dual-Write During Checkpointing

When your ingestion pipeline triggers a checkpoint at version C:

```rust
async fn write_checkpoint_and_sidecar(state: &TableState, version: i64) {
    // 1. Standard V1 checkpoint (for all readers)
    write_v1_checkpoint(state, version).await;
    
    // 2. Optimized sidecar (for enhanced readers)
    let sidecar = state
        .to_columnar_batch()          // Convert to Arrow with native stat columns
        .sort_by(&["_event_hour"])     // Sort by partition for row-group pruning
        .write_parquet(
            &format!("_delta_log/_sidecars/v{}.parquet", version),
            ParquetWriterOptions {
                row_group_size: 10_000,    // ~10K files per row group
                compression: Compression::ZSTD,
            },
        )
        .await;
}
```

### 4.2 Sidecar Schema

The sidecar "explodes" the V1 JSON stats into native Parquet columns:

**V1 Checkpoint Row (current):**

| Column | Type | Size per file |
|--------|------|---------------|
| `add.path` | String | ~100 bytes |
| `add.size` | Long | 8 bytes |
| `add.partitionValues` | Map<String, String> | ~50 bytes |
| `add.stats` | **String (JSON!)** | **1-10 KB** |
| `add.deletionVector` | Struct (nullable) | ~50 bytes |

**Sidecar Row (proposed):**

| Column | Type | Size per file | Notes |
|--------|------|---------------|-------|
| `path` | String | ~100 bytes | File path |
| `size` | Int64 | 8 bytes | File size in bytes |
| `modification_time` | Int64 | 8 bytes | Epoch millis |
| `_event_hour` | String | 13 bytes | e.g., "2026020114" (partition) |
| `num_records` | Int64 | 8 bytes | Row count |
| `min.<col>` | (matches column type) | varies | Per-column min value (native Parquet type) |
| `max.<col>` | (matches column type) | varies | Per-column max value (native Parquet type) |
| `null_count.<col>` | Int64 | 8 bytes | Per-column null count |
| `dv.storageType` | String (nullable) | ~2 bytes | DV storage type (e.g., "u" for UUID) |
| `dv.pathOrInlineDv` | String (nullable) | ~40 bytes | DV file path or inline data |
| `dv.offset` | Int32 (nullable) | 4 bytes | Offset within DV file |
| `dv.sizeInBytes` | Int32 (nullable) | 4 bytes | Size of DV data |
| `dv.cardinality` | Int64 (nullable) | 8 bytes | Number of deleted rows |

> **Note:** The `min.<col>`, `max.<col>`, and `null_count.<col>` rows above are repeated for **every column that has statistics** in the original V1 checkpoint's JSON stats. For example, a table with 50 stats-collected columns would have 150 stat columns in the sidecar (50 × min/max/null_count). The key advantage over V1 is that each stat is a **native Parquet column** (Timestamp, String, Int64, etc.) rather than buried inside a JSON string — enabling column projection: a query on `WHERE timestamp > X` only reads `min.timestamp` and `max.timestamp`, skipping all other stat columns entirely.

**Memory savings per file: ~10 KB (JSON) → ~250 bytes (native) = 40x reduction** (varies with number of stats-collected columns)

For 10M files: **100 GB (V1 parsed) → 2 GB (sidecar on disk) → ~50 MB in memory per row group batch**

### 4.3 Sorting Guarantee

The sidecar is **sorted by `_event_hour`** before writing. This ensures:

```
Row Group 0:  _event_hour min="2026020100" max="2026020103"
Row Group 1:  _event_hour min="2026020104" max="2026020107"
Row Group 2:  _event_hour min="2026020108" max="2026020111"  ← Query: hour=10? Only this RG!
Row Group 3:  _event_hour min="2026020112" max="2026020115"
...
```

Without sorting, every row group would have `min="2026010100" max="2026021023"` and **no pruning is possible**.

### 4.4 Sidecar Lifecycle

| Event | Sidecar Action |
|-------|----------------|
| Checkpoint at version C | Write `_sidecars/vC.parquet` |
| Next checkpoint at C+10 | Write `_sidecars/v{C+10}.parquet` |
| VACUUM | Delete sidecars older than retention |
| Schema change | New sidecar reflects new columns |

Only the **latest sidecar** is needed. Old sidecars can be cleaned up.

### 4.5 Manifest File: Pre-Extracted Footer for Zero-Latency Discovery

#### The Problem: Parquet Footer Fetch Latency

When opening the sidecar Parquet for the first time, the reader must perform **2 sequential HTTP requests** to Azure Blob Storage before any data can flow:

1. `GET` with `Range: bytes=-8` → read the last 8 bytes to learn the footer length
2. `GET` with `Range: bytes=X-Y` → read the footer (Thrift-encoded row group metadata)

Each round-trip to Azure Blob is ~20-50ms, adding **40-100ms of pure overhead** before the reader even knows the sidecar's structure.

#### The Solution: Write a Manifest Alongside the Sidecar

At checkpoint time, the ingestion pipeline writes a **small manifest file** containing the pre-extracted row group index:

```
_delta_log/_sidecars/
├── v1000.parquet              ← Sidecar data (row groups with native stats)
└── v1000.manifest.json        ← Pre-extracted row group index (~1-10 KB)
```

**Manifest format:**

```json
{
  "version": 1000,
  "sidecar_file": "v1000.parquet",
  "sidecar_size_bytes": 2147483648,
  "num_row_groups": 1000,
  "num_files": 10000000,
  "schema_hash": "a1b2c3d4",
  "row_groups": [
    {
      "index": 0,
      "byte_offset": 0,
      "byte_length": 524288,
      "num_rows": 10000,
      "event_hour_min": "2026010100",
      "event_hour_max": "2026010123"
    },
    {
      "index": 1,
      "byte_offset": 524288,
      "byte_length": 524288,
      "num_rows": 10000,
      "event_hour_min": "2026010200",
      "event_hour_max": "2026010223"
    }
  ]
}
```

#### How the Manifest Changes the Pipeline

**Without manifest** (Parquet footer path):

```
Step 2: Parallel Discovery
┌──────────────────────┐  ┌───────────────────────────────────────┐
│ Task A: Fetch JSONs  │  │ Task B: HEAD sidecar (50ms)           │
│ C+1..V in parallel   │  │ Then: GET footer length (30ms)        │
│                      │  │ Then: GET footer content (30ms)       │
│                      │  │ Then: Deserialize Thrift footer (5ms) │
└──────────────────────┘  └───────────────────────────────────────┘
                            Total sidecar setup: ~115ms (3 serial requests)
```

**With manifest** (single GET path):

```
Step 2: Parallel Discovery
┌──────────────────────┐  ┌───────────────────────────────────────┐
│ Task A: Fetch JSONs  │  │ Task B: GET manifest.json (30ms)      │
│ C+1..V in parallel   │  │ Already contains all row group info!  │
│                      │  │ Parse JSON (~1ms)                     │
└──────────────────────┘  └───────────────────────────────────────┘
                            Total sidecar setup: ~31ms (1 request)
```

#### Performance Analysis

| Metric | Without Manifest | With Manifest | Savings |
|--------|-----------------|---------------|---------|
| **HTTP requests for sidecar setup** | 3 sequential (HEAD + footer length + footer) | 1 (GET manifest) | 2 fewer requests |
| **Sidecar setup latency** | ~115ms (3 × 30-50ms) | ~31ms (1 × 30ms + 1ms parse) | **~84ms saved** (~73% reduction) |
| **Sidecar setup data transferred** | ~200 KB (Thrift footer) | ~5 KB (JSON manifest) | **~195 KB saved** (~97% reduction) |
| **Footer parsing CPU** | Thrift deserialization (complex) | JSON parse (trivial) | Negligible in both cases |
| **Sidecar discovery** | HEAD request needed first | Manifest existence = sidecar exists | 1 fewer request |
| **Time to first row group read** | ~165ms (setup + first RG fetch) | ~81ms (manifest + first RG fetch) | **~84ms faster** |

**For the full pipeline (time to first file yielded):**

| Step | Without Manifest | With Manifest |
|------|-----------------|---------------|
| Version discovery | 50ms | 50ms |
| Parallel fetch (JSONs + sidecar setup) | max(100ms JSONs, 115ms footer) = 115ms | max(100ms JSONs, 31ms manifest) = 100ms |
| Build tombstones + yield JSON adds | 10ms | 10ms |
| First sidecar row group read | 50ms | 50ms |
| **Total to first file** | **~225ms** | **~210ms** |

The manifest saves ~15ms on the critical path when JSONs take longer than the manifest fetch. However, the **real win** is in robustness and simplicity:

#### Why the Manifest Matters Beyond Latency

1. **Eliminates Parquet footer parsing entirely** — No Thrift deserialization, no binary footer parsing code path for the sidecar. The reader goes straight from manifest JSON to targeted row group reads.

2. **Row-group pruning without opening the file** — The manifest contains `event_hour_min`/`event_hour_max` per row group. The reader can determine which row groups to read before making any request to the sidecar Parquet. For a "last 1 hour" query on a sidecar with 1,000 row groups, this avoids opening the sidecar file entirely if the manifest shows no matching row groups (e.g., stale checkpoint).

3. **Manifest doubles as sidecar discovery** — If the manifest file exists, the sidecar exists. No separate HEAD request needed. If the manifest GET returns 404, fall back to V1 checkpoint.

4. **Future extensibility** — The manifest can carry additional metadata without changing the Parquet format:
   - Column-level statistics summaries
   - Sidecar schema fingerprint (detect schema changes)
   - Compression codec info
   - Custom ingestion metadata

#### Manifest Lifecycle

The manifest is written atomically alongside the sidecar:

```rust
async fn write_checkpoint_and_sidecar(state: &TableState, version: i64) {
    // 1. Standard V1 checkpoint
    write_v1_checkpoint(state, version).await;
    
    // 2. Write sidecar Parquet
    let sidecar_metadata = write_sidecar_parquet(state, version).await;
    
    // 3. Write manifest (derived from sidecar's row group metadata)
    let manifest = Manifest {
        version,
        sidecar_file: format!("v{}.parquet", version),
        sidecar_size_bytes: sidecar_metadata.file_size,
        num_row_groups: sidecar_metadata.row_groups.len(),
        num_files: state.num_active_files(),
        row_groups: sidecar_metadata.row_groups.iter().map(|rg| {
            ManifestRowGroup {
                index: rg.index,
                byte_offset: rg.byte_offset,
                byte_length: rg.compressed_size,
                num_rows: rg.num_rows,
                event_hour_min: rg.column("_event_hour").min_value(),
                event_hour_max: rg.column("_event_hour").max_value(),
            }
        }).collect(),
    };
    
    write_json(
        &format!("_delta_log/_sidecars/v{}.manifest.json", version),
        &manifest,
    ).await;
}
```

| Event | Manifest Action |
|-------|-----------------|
| Checkpoint at version C | Write `v{C}.manifest.json` alongside `v{C}.parquet` |
| VACUUM | Delete old manifests alongside old sidecars |
| Manifest missing but sidecar exists | Fall back to Parquet footer path (2 extra requests) |
| Both missing | Fall back to V1 checkpoint |

---

## 5. Component 2: delta-rs Enhancement — Streaming Metadata Provider

### 5.1 The Sidecar-Aware Metadata Loader

The reader uses a **3-tier fallback chain** to discover the best available metadata source:

```
┌─────────────────────────────────────────────────────────────┐
│ Tier 1: GET manifest.json                                   │
│ _sidecars/vC.manifest.json → 200 OK?                        │
│   YES → SidecarWithManifest (fastest: 1 request, no footer) │
│   NO (404) ↓                                                │
├─────────────────────────────────────────────────────────────┤
│ Tier 2: HEAD sidecar.parquet                                │
│ _sidecars/vC.parquet → 200 OK?                              │
│   YES → SidecarWithFooter (read Parquet footer, +2 requests)│
│   NO (404) ↓                                                │
├─────────────────────────────────────────────────────────────┤
│ Tier 3: V1 Checkpoint                                       │
│ Standard checkpoint path (legacy, no sidecar optimization)  │
└─────────────────────────────────────────────────────────────┘
```

```rust
/// A metadata provider that prioritizes sidecar files over V1 checkpoints
pub struct SidecarAwareProvider {
    store: Arc<dyn ObjectStore>,
    table_path: Path,
}

impl SidecarAwareProvider {
    /// Discover the best metadata source using 3-tier fallback:
    ///   1. Manifest + sidecar (fastest — no Parquet footer parsing)
    ///   2. Sidecar only (fall back to Parquet footer — +2 HTTP requests)
    ///   3. V1 checkpoint (standard path — no sidecar available)
    async fn discover(&self, checkpoint_version: i64) -> MetadataSource {
        let base = format!("{}/_delta_log/_sidecars/v{}", self.table_path, checkpoint_version);
        let manifest_path = format!("{}.manifest.json", base);
        let sidecar_path = format!("{}.parquet", base);
        
        // Tier 1: Try manifest (single GET, ~5 KB, ~30ms)
        if let Ok(manifest_bytes) = self.store.get(&manifest_path.into()).await {
            let manifest: Manifest = serde_json::from_slice(&manifest_bytes.bytes().await?)?;
            return MetadataSource::SidecarWithManifest(manifest, sidecar_path);
        }
        
        // Tier 2: Try sidecar without manifest (HEAD + footer reads, ~115ms)
        if let Ok(_) = self.store.head(&sidecar_path.into()).await {
            return MetadataSource::SidecarWithFooter(sidecar_path);
        }
        
        // Tier 3: Fall back to V1 checkpoint
        MetadataSource::V1Checkpoint(checkpoint_version)
    }
}

enum MetadataSource {
    /// Fastest: manifest provides row group index, skip Parquet footer entirely
    SidecarWithManifest(Manifest, String),
    /// Sidecar exists but no manifest: read Parquet footer (+2 HTTP requests)
    SidecarWithFooter(String),
    /// No sidecar: standard V1 checkpoint path
    V1Checkpoint(i64),
}
```

**When does each tier apply?**

| Scenario | Tier | Latency | When this happens |
|----------|------|---------|-------------------|
| Manifest + sidecar | Tier 1 | ~31ms | Normal operation (ingestion writes both) |
| Sidecar only | Tier 2 | ~115ms | Older sidecars written before manifest support was added |
| V1 checkpoint | Tier 3 | ~200ms+ | Tables without sidecar (not using enhanced ingestion) |

### 5.2 Parallel Log Tailing

Instead of fetching JSON commits sequentially:

```rust
/// Fetch all JSON commits since checkpoint in parallel.
/// Returns:
///   - tombstones: files removed and NOT re-added (truly deleted)
///   - overrides: files removed AND re-added (DV update or rewrite) — must suppress sidecar version
///   - recent_adds: all add actions from commits (with latest DV metadata if present)
async fn fetch_commits_parallel(
    store: &dyn ObjectStore,
    from_version: i64,   // checkpoint + 1
    to_version: i64,     // current head
) -> (HashSet<String>, HashSet<String>, Vec<AddAction>) {
    let mut tombstones: HashSet<String> = HashSet::new();
    let mut overrides: HashSet<String> = HashSet::new();
    let mut recent_adds: Vec<AddAction> = Vec::new();
    
    // Fan-out: fetch all JSONs concurrently
    let futures: Vec<_> = (from_version..=to_version)
        .map(|v| store.get(&commit_path(v)))
        .collect();
    
    let results = futures::future::join_all(futures).await;
    
    // Process in version order for correct reconciliation
    for (version, result) in results.into_iter().enumerate() {
        let (commit_removes, commit_adds) = process_commit(parse_commit(result?)?);
        
        for path in commit_removes {
            tombstones.insert(path);
        }
        for add in commit_adds {
            if tombstones.remove(&add.path) {
                // This add overrides a previous remove for the same path.
                // This happens during DV updates (remove old add, re-add with new DV)
                // or OPTIMIZE rewrites. The sidecar has the stale version — must suppress it.
                overrides.insert(add.path.clone());
            }
            recent_adds.push(add);
        }
    }
    
    (tombstones, overrides, recent_adds)
}
```

For 10 commits since checkpoint, this takes **1 round-trip** (~50-200ms) instead of 10 sequential fetches.

> **DV note:** The `AddAction` struct carries an optional `deletion_vector` field. When DVs are enabled and rows are deleted, the protocol issues a `remove` for the old `add` and a new `add` for the same file path with an updated DV. The algorithm tracks these as **overrides** (not tombstones) so the stale sidecar entry is suppressed and the latest `add` with the correct DV is yielded from the JSON commits.

### 5.3 The Streaming Pipeline

```rust
/// Stream files: yield recent adds immediately, then stream sidecar
pub fn stream_files(
    &self,
    predicate: Option<&Predicate>,
    limit: Option<usize>,
) -> impl Stream<Item = DeltaResult<FileInfo>> {
    async_stream::stream! {
        let checkpoint_version = self.read_last_checkpoint().await?;
        let head_version = self.find_head_version().await?;
        
        // Step 1: Parallel fetch of all JSONs since checkpoint
        // Returns three sets:
        //   tombstones — truly removed files (not re-added)
        //   overrides  — files removed AND re-added (DV update / rewrite)
        //   recent_adds — all add actions (with latest DV if applicable)
        let (tombstones, overrides, recent_adds) = fetch_commits_parallel(
            &self.store,
            checkpoint_version + 1,
            head_version,
        ).await?;
        
        // Combine for sidecar filtering: skip files that are tombstoned OR overridden
        let sidecar_exclude: HashSet<&str> = tombstones.iter()
            .chain(overrides.iter())
            .map(|s| s.as_str())
            .collect();
        
        let mut emitted = 0usize;
        
        // Step 2: Yield recent adds IMMEDIATELY
        // These include DV-updated files (overrides) with the latest DV metadata.
        // The query engine can start working while we stream the sidecar.
        for add in recent_adds.iter() {
            if let Some(p) = predicate { if !p.matches(add) { continue; } }
            
            yield Ok(add.into());
            emitted += 1;
            if limit.map_or(false, |l| emitted >= l) { return; }
        }
        
        // Step 3: Stream sidecar (or V1 checkpoint as fallback)
        let source = self.discover(checkpoint_version).await;
        let reader = match source {
            MetadataSource::Sidecar(path) => {
                open_parquet_async(&self.store, &path).await?
            }
            MetadataSource::V1Checkpoint(v) => {
                open_checkpoint_async(&self.store, v).await?
            }
        };
        
        // Step 4: Stream row groups with filtering
        for rg_idx in 0..reader.num_row_groups() {
            // Row-group level pruning (only works if sidecar is sorted)
            if let Some(p) = predicate {
                if !p.matches_row_group_stats(reader.row_group_metadata(rg_idx)) {
                    continue;  // Skip entire row group!
                }
            }
            
            let batch = reader.read_row_group(rg_idx).await?;
            
            for file in batch.iter() {
                // Skip files that were removed OR overridden (DV update) in commits C+1..V
                if sidecar_exclude.contains(file.path.as_str()) { continue; }
                if let Some(p) = predicate { if !p.matches(file) { continue; } }
                
                yield Ok(file.into());
                emitted += 1;
                if limit.map_or(false, |l| emitted >= l) { return; }
            }
            // batch memory released here
        }
    }
}
```

### 5.4 Column Projection

When streaming the sidecar, delta-rs only reads the columns it needs:

| Query | Columns Read from Sidecar | Columns Skipped |
|-------|--------------------------|-----------------|
| `SELECT * FROM t LIMIT 100` | `path`, `size` | All stat columns |
| `WHERE timestamp > X` | `path`, `size`, `min.timestamp`, `max.timestamp` | Other stat columns |
| `WHERE device_id = 'abc'` | `path`, `size`, `min.device_id`, `max.device_id` | Other stat columns |

With V1 JSON stats, you **must read the entire JSON string** to extract any one stat. With the sidecar, you read only the columns you need.

---

## 6. Component 3: Kusto Integration

### 6.1 Contribution Strategy


| Change | Where | Impact |
|--------|-------|--------|
| `SidecarAwareProvider` trait | delta-rs core | Enables sidecar discovery |
| Parallel log tailing | delta-rs core | Faster commit fetching |
| Streaming metadata API | delta-rs core | Non-blocking file discovery |
| Sidecar integration | Kusto delta connector | Uses sidecar when available |

### 6.2 Kusto-Specific Benefits

```
Current Kusto Query Flow:
  1. delta-rs loads full V1 checkpoint (10 GB) ──── 60+ seconds, OOM risk
  2. Parse JSON stats for every file ──────────── CPU-bound
  3. Build file list ──────────────────────────── Memory-bound
  4. Pass to Kusto engine ─────────────────────── Finally starts

With Sidecar:
  1. delta-rs finds sidecar (HEAD request) ────── 50 ms
  2. Fetch 10 JSONs in parallel ───────────────── 100 ms
  3. Stream sidecar row groups to engine ──────── First files in 200 ms
  4. Kusto starts reading Parquet in parallel ─── Immediate
```

### 6.3 Backward Compatibility

- Kusto sees the **same V1 checkpoint** if sidecar is missing
- The sidecar is in `_sidecars/` subfolder — invisible to legacy readers
- No protocol version bump required
- No `metaData` or `protocol` action changes

---

## 7. The Query Pipeline

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ Step 1: Version Discovery                                           │
│ ─────────────────────────────────────────────────────────────────── │
│ Read _last_checkpoint → checkpoint version C                        │
│ List _delta_log/*.json → head version V                             │
│ Pin snapshot at version V (all subsequent reads ≤ V)                │
│                                                              ~50 ms │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 2: Parallel Discovery (concurrent)                             │
│ ─────────────────────────────────────────────────────────────────── │
│                                                                     │
│ ┌─────────────────────────┐  ┌─────────────────────────────────┐    │
│ │ Task A: Fetch JSONs     │  │ Task B: GET manifest            │    │
│ │ C+1, C+2, ..., V        │  │ _sidecars/vC.manifest.json      │    │
│ │ (all in parallel)       │  │ (~5 KB, 1 request, ~30ms)       │    │
│ └─────────────────────────┘  └─────────────────────────────────┘    │
│                                                              ~100ms │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 3: Build Tombstone + Override Sets                             │
│ ─────────────────────────────────────────────────────────────────── │
│ Extract `remove` paths → tombstones (truly deleted files)           │
│ Detect remove+re-add → overrides (DV updates / rewrites)            │
│ Both sets used to suppress stale sidecar entries                    │
│ (In ADX context: both sets almost always EMPTY — appends only)      │
│                                                               ~10ms │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 4: Yield Recent Adds                                           │
│ ─────────────────────────────────────────────────────────────────── │
│ Extract `add` actions from JSONs                                    │
│ Filter by predicate + tombstone set                                 │
│ YIELD to query engine immediately                                   │
│ ──► ENGINE STARTS WORKING HERE ◄──                                  │
│                                                                ~0ms │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Step 5: Stream Sidecar (or V1 checkpoint fallback)                  │
│ ─────────────────────────────────────────────────────────────────── │
│ FOR each row group in sidecar:                                      │
│   • Check row-group stats against predicate → SKIP if no match      │
│   • Read row group (column projection: only needed columns)         │
│   • Filter by tombstone + override sets                             │
│   • YIELD matching files to engine                                  │
│   • Release row-group memory                                        │
│   • If LIMIT reached → STOP                                         │
│                                                                     │
│ Memory at any point: O(tombstones) + O(1 row group) ≈ 50 MB         │
└─────────────────────────────────────────────────────────────────────┘
```

### Critical Requirement: Tombstones Must Be Complete Before Sidecar Streaming

The tombstone set (removes from JSONs) **must be fully built** before streaming the sidecar. Otherwise we might yield files that have been removed. However:

1. JSONs are tiny (KB each) and fetched in parallel — takes ~100ms
2. The sidecar is large and needs Parquet footer parsing — takes ~200ms
3. **Natural ordering**: tombstones are ready before the sidecar stream starts

For the `add` actions from JSONs (Step 4), these are safe to yield immediately because they come from commits **after** the checkpoint — they cannot be in the sidecar.

### Parallelism Model

The pipeline achieves **four levels of parallelism** to maximize throughput and minimize latency:

```
Time ──────────────────────────────────────────────────────────────►

Level 1: Concurrent I/O (network)
┌─────────────────────────────────────────────────────────────────┐
│ ┌─── GET 101.json ───┐                                          │
│ ┌─── GET 102.json ───┐  All JSONs fetched in ONE round-trip     │
│ ┌─── GET 103.json ───┐  (concurrent HTTP requests)              │
│ ┌─── GET manifest ───┐  Manifest fetched in parallel            │
└─────────────────────────────────────────────────────────────────┘

Level 2: Pipelined Discovery → Execution
┌─────────────────────────────────────────────────────────────────┐
│ [Build tombstones]                                              │
│ [Yield JSON adds]──►[Engine reads Parquet 1, 2, 3...]           │
│ [Stream sidecar RG1]──►[Engine reads Parquet 4, 5, 6...]        │
│ [Stream sidecar RG2]──►[Engine reads Parquet 7, 8, 9...]        │
│                                                                 │
│ Engine does NOT wait for all files — it starts on first batch   │
└─────────────────────────────────────────────────────────────────┘

Level 3: Engine-Side Parallelism (Kusto)
┌─────────────────────────────────────────────────────────────────┐
│ As file paths arrive, Kusto distributes reads across nodes:     │
│                                                                 │
│ Node 1: ──[Read part-0001.parquet]──[Read part-0004.parquet]──  │
│ Node 2: ──[Read part-0002.parquet]──[Read part-0005.parquet]──  │
│ Node 3: ──[Read part-0003.parquet]──[Read part-0006.parquet]──  │
│                                                                 │
│ Full cluster utilization from the first yielded file            │
└─────────────────────────────────────────────────────────────────┘

Level 4: Early Termination
┌─────────────────────────────────────────────────────────────────┐
│ For `| take 100` queries:                                       │
│                                                                 │
│ [Yield 3 files from JSON adds]                                  │
│ [Stream sidecar RG1 → yield 97 files] → LIMIT reached → STOP    │
│ [Sidecar RG2..N never read]                                     │
│                                                                 │
│ Total metadata read: ~1 row group + 10 tiny JSONs               │
└─────────────────────────────────────────────────────────────────┘
```

**Comparison: Sequential vs. Pipelined**

```
Sequential (Current):
  [==== Load ALL metadata (60s) ====][=== Query (5s) ===]
  Total: 65 seconds, 10 GB memory

Pipelined (Sidecar):
  [JSON fetch (0.1s)]
  [Yield adds]──►[Query starts (0.2s)]
  [Stream RG1]──►[Query continues...]
  [Stream RG2]──►[Query continues...]
  Total: ~5-10 seconds (overlapped), 50 MB memory
```

---

## 8. Action Frequency Analysis (ADX Context)

### Your Workload Profile

Since these are telemetry/log tables ingested to Azure Blob and queried by Kusto:

| Action | Frequency | Reason | Impact on Design |
|--------|-----------|--------|------------------|
| `add` | **99%+** | Continuous ingestion of telemetry | Sidecar is the primary metadata source |
| `remove` | **<1%** | Only during OPTIMIZE/compaction | Tombstone set is almost always empty |
| `commitInfo` | Every commit | Standard audit trail | Ignored by reader |
| `txn` | Common | Idempotency tracking for streaming sources | Skipped — no effect on file state |
| `metaData` | Rare | Schema evolution | Skipped — sidecar is rebuilt at next checkpoint with new schema |
| `protocol` | Rare | Feature enablement (DVs, column mapping, etc.) | Skipped — sidecar is rebuilt at next checkpoint |
| `deletionVector` | Possible | Attached to `add` actions when DVs are enabled | Carried through — DV metadata included in sidecar and yielded files |

### Implications

1. **Tombstone set is negligible** — O(removes) ≈ O(0) in practice
2. **All Delta actions are valid** — no restrictions on features used by the table
3. **Only `add` and `remove` affect file discovery** — all other actions are informational or captured at checkpoint time
4. **Schema evolution is safe** — the sidecar is rebuilt at each checkpoint, reflecting the latest schema
5. **DVs are supported** — DV metadata is carried in the sidecar and in JSON `add` actions, passed through to the engine

### Action Handling in the Sidecar Pipeline

The sidecar pipeline only needs to process `add` and `remove` actions from the JSON commits between checkpoints. All other actions are **safely skipped** because:

- **`metaData`** — Schema is captured in the sidecar at checkpoint time. Mid-checkpoint schema changes are reflected in the next sidecar.
- **`protocol`** — Protocol features affect how the engine reads data files, not how the sidecar streams file metadata. The engine handles protocol compliance.
- **`commitInfo`** — Audit metadata with no effect on file state.
- **`txn`** — Idempotency markers with no effect on file state.
- **Deletion Vectors** — Not a separate action type; they are attached to `add` actions as the `deletionVector` field. The pipeline carries this field through to the engine.

| Action | Handling | Reason |
|--------|----------|--------|
| `add` | ✅ Process | Core action — file discovery (includes DV metadata if present) |
| `remove` | ✅ Process | Core action — tombstone tracking |
| `commitInfo` | ✅ Skip | No effect on file state |
| `txn` | ✅ Skip | No effect on file state |
| `metaData` | ✅ Skip | Captured at checkpoint time in sidecar |
| `protocol` | ✅ Skip | Engine handles protocol compliance |
| Any other | ✅ Skip | Only `add`/`remove` affect file discovery |

---

## 9. Hourly Virtual Partitioning

### The Problem Without Partitions

Without partitions, a query for `WHERE timestamp > ago(1h)` must:
1. Load metadata for **all** files (even files from months ago)
2. Parse stats for every file to check timestamp ranges
3. Discard 99%+ of files after checking

### The Solution: Time-Aligned File Writing

During ingestion, add a **generated partition column** based on a configurable time boundary:

```sql
_event_hour = FORMAT(date_trunc('hour', event_timestamp), 'yyyyMMddHH')
-- Example: "2026021014" for Feb 10, 2026 at 2:00 PM
```

> **Note:** This document uses **hour** as the time boundary throughout, but the approach works with any granularity — **month, day, hour, minute, or second**. The choice depends on your data volume and query patterns:
> 
> | Boundary | Partition key format | Best for |
> |----------|---------------------|----------|
> | Month | `yyyyMM` | Low-volume tables, long-range queries |
> | Day | `yyyyMMdd` | Medium-volume tables |
> | **Hour** | `yyyyMMddHH` | **High-volume telemetry (recommended)** |
> | Minute | `yyyyMMddHHmm` | Very high-throughput streams |
> | Second | `yyyyMMddHHmmss` | Ultra-high-frequency event streams |
> 
> The column name, format string, and `date_trunc` granularity are all configurable in the ingestion pipeline. The sidecar sorting and row-group pruning work identically regardless of the chosen boundary.

#### File Splitting Rule

**Every Parquet file must contain rows from exactly one `_event_hour`.** The ingestion pipeline enforces this by partitioning rows before writing.

**During ingestion**, when writing a batch of rows:

1. Compute `_event_hour` for each row: `format(date_trunc('hour', event_timestamp), 'yyyyMMddHH')`
2. **Group rows by `_event_hour`**
3. Write each group to a separate Parquet file under the corresponding partition directory

**Example**: A batch arrives with 10,000 rows spanning 11:45 PM to 12:15 AM:

```
Incoming batch: 10,000 rows with timestamps from 23:45 to 00:15

Split by _event_hour:
  5,000 rows → _event_hour = "2026021023" → _event_hour=2026021023/part-XXXX.parquet
  5,000 rows → _event_hour = "2026021100" → _event_hour=2026021100/part-YYYY.parquet
```

**Size-based splitting still applies within each hour**: if one hour accumulates more than 128 MB of data, it is split into multiple files — all under the same `_event_hour` partition.

```
Effective rule:  Write new file when (size >= 128 MB) OR (row._event_hour != current_file._event_hour)
```

**Why this matters**: If a single file contains rows from multiple hours:
- The file's `min.timestamp` and `max.timestamp` would span multiple hours
- It cannot be assigned to a single `_event_hour` partition value
- The sidecar's row-group pruning by `_event_hour` becomes **ineffective** — the file must be included in every hour range it touches
- A query for "last 1 hour" would pull in files from adjacent hours, defeating the purpose of partitioning

#### Physical Layout

```
table_root/
├── _event_hour=2026021013/
│   ├── part-0001.parquet    ← Only 1:00-1:59 PM data
│   └── part-0002.parquet
├── _event_hour=2026021014/
│   ├── part-0001.parquet    ← Only 2:00-2:59 PM data
│   └── part-0002.parquet
└── _delta_log/
    └── _sidecars/
        └── v1000.parquet     ← Sorted by _event_hour
```

### How This Enables Row-Group Pruning

The sidecar is sorted by `_event_hour`. Each row group covers a contiguous range of hours:

```
Sidecar Parquet Footer:
  Row Group 0: _event_hour [2026020100, 2026020123]  ← Skip (not today)
  Row Group 1: _event_hour [2026020200, 2026020223]  ← Skip
  ...
  Row Group 9: _event_hour [2026021000, 2026021023]  ← READ (today!)
```

For a query on "last 2 hours": read **1 row group** instead of all row groups.

### Memory Impact

| Query Window | Without Partition | With Hourly Partition |
|--------------|-------------------|----------------------|
| Last 1 hour | Load ALL metadata (10M files) | Load 1 row group (~10K files) |
| Last 24 hours | Load ALL metadata | Load 1-3 row groups |
| Last 30 days | Load ALL metadata | Load ~30 row groups |
| Full table | Load ALL metadata | Load all row groups (same) |

---

## 10. Memory Analysis

### Comparison Table

| Approach | Memory Model | 10M Files | 100M Files |
|----------|-------------|-----------|------------|
| **Current V1** | O(all files × JSON expansion) | 10-50 GB | 100-500 GB (OOM) |
| **Lazy Iterator (no sidecar)** | O(removes) + O(1 RG of checkpoint) | ~150 MB | ~150 MB |
| **Sidecar (no partition)** | O(removes) + O(1 RG of sidecar) | ~50 MB | ~50 MB |
| **Sidecar + hourly partition** | O(removes) + O(matching RGs only) | ~5 MB | ~5 MB |

### Why Sidecar Beats V1 Checkpoint Streaming

Even if you stream the V1 checkpoint row-by-row, each row contains the **JSON stats string** that must be parsed. The sidecar eliminates JSON parsing entirely:

| Operation | V1 Checkpoint | Sidecar |
|-----------|--------------|---------|
| Read row | ~10 KB (JSON stats included) | ~200 bytes (native columns) |
| Parse stats | JSON deserialization required | Already native — zero cost |
| Column projection | Must read entire JSON to get one stat | Read only needed columns |
| Row-group skip | No sort guarantee → read all | Sorted → skip non-matching |

---

## 11. Correctness & Safety

### 11.1 Tombstone Completeness

**Requirement**: All `remove` actions from commits C+1..V must be collected before streaming the sidecar.

**Guarantee**: Parallel JSON fetching completes before sidecar streaming begins (Step 3 before Step 5 in the pipeline).

**Edge case**: If a JSON commit fails to download, the entire query must fail — partial tombstone sets risk returning deleted files.

### 11.2 Version Pinning

**Requirement**: The snapshot version V must be determined once at query start.

**Guarantee**: 
- V = max version number from listing `_delta_log/*.json`
- All subsequent reads only include commits ≤ V
- New commits after V are ignored for this query

### 11.3 Sidecar Version Alignment

**Requirement**: The sidecar must correspond to the same checkpoint version.

**Guarantee**: Sidecar filename contains the version (`v1000.parquet`). The reader verifies that the sidecar version matches the checkpoint version from `_last_checkpoint`.

**Failure mode**: If sidecar version doesn't match → fall back to V1 checkpoint.

### 11.4 Action Handling

When scanning JSON commits, only `add` and `remove` actions affect file discovery. All other standard Delta actions (`metaData`, `protocol`, `commitInfo`, `txn`, etc.) are safely skipped — they do not change which files are active.

This is correct because:
- The sidecar represents the **complete file state at checkpoint version C**, already accounting for all actions up to C
- Only commits C+1..V can change the active file set, and they do so exclusively through `add` and `remove` actions
- Schema/protocol changes in commits C+1..V will be reflected in the **next** sidecar when a new checkpoint is written
- Deletion vector metadata is carried as part of the `add` action's `deletionVector` field, not as a separate action

### 11.5 Add Action in JSON vs. Sidecar Deduplication

Files added in commits C+1..V will NOT appear in the sidecar (which represents state at version C). Therefore:
- Files from JSON adds and files from sidecar are **disjoint sets**
- No deduplication needed
- This is a natural property of the Delta log structure

### 11.6 Same-Commit Add/Remove Atomicity

Within a single commit, a file can be both removed (old version) and added (new version) for the same path. This happens during:
- **OPTIMIZE**: remove old small files, add new compacted file
- **DV updates**: remove old `add` for `file_A`, re-add `file_A` with updated deletion vector

The `process_commit` function handles this by letting adds override removes for the same path within the same commit:

```rust
fn process_commit(actions: &[Action]) -> (Vec<String>, Vec<AddAction>) {
    let mut removes: HashSet<String> = HashSet::new();
    let mut adds: Vec<AddAction> = Vec::new();
    
    for action in actions {
        match action {
            Action::Remove { path, .. } => { removes.insert(path.clone()); }
            Action::Add(add) => { adds.push(add.clone()); }
            _ => {}
        }
    }
    
    // Same-commit adds override same-commit removes
    for add in &adds {
        removes.remove(&add.path);
    }
    
    (removes.into_iter().collect(), adds)
}
```

The per-commit removes returned by this function are then fed into `fetch_commits_parallel` (Section 5.2), which tracks the **override set** — files that were removed in one commit and re-added in a later (or same) commit. This ensures stale sidecar entries are suppressed.

### 11.7 Deletion Vector Update Correctness

When DVs are enabled and rows are deleted from an existing file, Delta Lake does NOT modify the file in place. Instead:

1. **`remove`** the old `add` action for `file_A.parquet` (with its old DV or no DV)
2. **`add`** a new action for the **same** `file_A.parquet` with an **updated DV**

This means `file_A.parquet` exists in both the **sidecar** (with the old/no DV) and in the **JSON commits** (with the new DV). Without special handling, the file would be yielded twice with conflicting DV metadata.

**The override set solves this:**

```
Checkpoint C: sidecar contains file_A with DV_v1 (3 deleted rows)

Commit C+3:
  remove(file_A)          ← tombstone
  add(file_A, DV_v2)      ← re-add with updated DV (5 deleted rows)

Algorithm:
  1. process_commit(C+3): same-commit override → remove is suppressed, add returned
  2. fetch_commits_parallel: tombstone.remove(file_A) → moved to overrides set
  3. Yield JSON adds: yield file_A with DV_v2 ✅ (latest)
  4. Stream sidecar: file_A is in sidecar_exclude (overrides) → SKIPPED ✅
```

**Result**: `file_A` is yielded exactly once, with the correct DV_v2. No DV merging is needed — the latest `add` action **replaces** the old one entirely.

---

## 12. API Design: Transparent Enhancement (No New API)

### Why No New API Is Needed

The sidecar optimization is implemented as an **internal enhancement** to delta-rs's existing `Snapshot` and `LogStore` components. No new public API or trait is required. Callers — including Kusto — continue to use the existing `DeltaTable::scan()` API unchanged.

**Rationale:**

| Concern | New API (rejected) | Transparent enhancement (chosen) |
|---------|-------------------|----------------------------------|
| Kusto integration | Kusto would need connector changes to call new API | **Zero changes** — Kusto calls the same delta-rs API |
| Code paths | Two parallel paths (old + new), doubles testing surface | **Single path** — sidecar is an internal optimization |
| delta-rs philosophy | Fragments the codebase with a parallel provider type | **Consistent** — fits into existing `Snapshot` → `DeltaScan` pipeline |
| Adoption barrier | Every consumer must opt in to the new API | **Automatic** — sidecar used when available, fallback when not |

### What Changes Internally

The sidecar replaces the "read checkpoint" step inside delta-rs's existing snapshot loading. The external API surface is unchanged:

```rust
// Caller code is IDENTICAL to today — no changes required:
let table = deltalake::open_table("abfss://container@account/table").await?;
let scan = table.scan()
    .with_predicate(predicate)
    .with_limit(100)
    .build()?;

// Internally, delta-rs now:
//   1. Discovers manifest → sidecar → V1 checkpoint (3-tier fallback)
//   2. Fetches JSON commits in parallel (instead of sequentially)
//   3. Streams file metadata with bounded memory (instead of materializing all)
//   4. Applies row-group pruning when sidecar is sorted by partition
```

| Internal Component | Current Behavior | With Sidecar | External API Impact |
|-------------------|-----------------|-------------|-------------------|
| `Snapshot::load()` | Reads V1 checkpoint + replays log | Checks for sidecar first, falls back to V1 | **None** |
| `LogStore::read_commit()` | Sequential JSON reads | Parallel JSON reads | **None** |
| `EagerSnapshot` | Materializes all files in memory | New `LazySnapshot` streams row groups | **None** — same `Snapshot` trait |
| `DeltaScan` | Receives full file list | Receives file stream | **None** — same scan API |

### Configuration (Optional)

The few new configuration options are added to the existing `DeltaTableBuilder` or as environment variables — no new types needed:

```rust
// Via DeltaTableBuilder (existing API):
let table = DeltaTableBuilder::from_uri("abfss://...")
    .with_option("delta.sidecar.enabled", "true")       // Enable sidecar discovery (default: true)
    .with_option("delta.commit.parallelism", "10")       // Parallel JSON fetches (default: 10)
    .build()
    .await?;

// Or via environment variables:
// DELTA_SIDECAR_ENABLED=true
// DELTA_COMMIT_PARALLELISM=10
```

| Option | Default | Description |
|--------|---------|-------------|
| `delta.sidecar.enabled` | `true` | Enable 3-tier sidecar discovery |
| `delta.commit.parallelism` | `10` | Max concurrent JSON commit fetches |
| `delta.sidecar.row_groups_per_batch` | `5` | Row groups to buffer per streaming batch (internal tuning) |

### One Addition: On-Demand Stats Retrieval

The only new **public** method is an optional convenience for retrieving stats for specific files from the sidecar:

```rust
impl DeltaTable {
    /// Retrieve column statistics for specific files from the sidecar.
    /// Falls back to V1 JSON stats if sidecar is not available.
    /// Useful for query engines that need stats after initial file discovery.
    pub async fn get_stats_for_files(
        &self,
        paths: &[String],
        stat_columns: &[String],
    ) -> DeltaResult<RecordBatch>;
}
```

This is a single method addition to the existing `DeltaTable` type — not a new trait or provider.

### Usage Examples

#### Example 1: Standard Query (Transparent — Zero Code Changes)

The most common usage. The sidecar optimization is completely invisible to the caller:

```rust
use deltalake::DeltaTableBuilder;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Open table — identical to today's API
    let table = DeltaTableBuilder::from_uri("abfss://container@account.dfs.core.windows.net/table")
        .with_storage_options(azure_credentials())
        .build()
        .await?;
    
    // Scan — identical to today's API
    // Internally: discovers sidecar → streams with bounded memory
    let scan = table.scan()
        .with_predicate(col("timestamp").gt(lit("2026-02-10T14:00:00Z")))
        .build()?;
    
    // Execute — Kusto calls this path
    // Files arrive as a stream: first from JSON commits, then from sidecar row groups
    let batches = scan.execute().await?;
    
    // Process results — engine started working from the first yielded file
    for batch in batches {
        println!("Got {} rows", batch?.num_rows());
    }
    
    Ok(())
}
```

**What happens behind the scenes:**
1. `build()` → 3-tier discovery: manifest → sidecar footer → V1 checkpoint
2. `execute()` → parallel JSON fetch, yield adds immediately, stream sidecar row groups
3. Memory never exceeds ~50 MB regardless of table size

#### Example 2: Low-Level Stream Consumption (Custom Engines)

For engines that want direct access to the file metadata stream for maximum control:

```rust
use deltalake::DeltaTableBuilder;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let table = DeltaTableBuilder::from_uri("abfss://container@account.dfs.core.windows.net/table")
        .with_storage_options(azure_credentials())
        .build()
        .await?;
    
    // Get a streaming iterator of active files
    // This is the lazy iterator — files are yielded one batch at a time
    let mut file_stream = table.snapshot()?.stream_active_files(
        Some(&predicate),   // Optional: partition/stats filter
        Some(1000),         // Optional: limit
    );
    
    let mut total_files = 0;
    let mut total_bytes = 0u64;
    
    // Consume the stream — each iteration reads one row group from sidecar
    // Memory: O(1 row group) at any time ≈ ~5 MB
    while let Some(file_result) = file_stream.next().await {
        let file = file_result?;
        
        // Schedule read immediately — don't wait for all files
        engine.schedule_parquet_read(&file.path, file.size);
        
        total_files += 1;
        total_bytes += file.size as u64;
        
        // Early termination: stop streaming if we have enough
        if total_files >= 1000 {
            break;  // Remaining sidecar row groups are never read
        }
    }
    // Stream dropped here — no cleanup needed
    
    println!("Scheduled {} files ({} GB)", total_files, total_bytes / 1_073_741_824);
    Ok(())
}
```

**Key streaming behaviors:**
- `next().await` returns the next batch of files (from JSON adds first, then sidecar row groups)
- Each call reads **one row group** from the sidecar (~10K files, ~5 MB)
- Previous row group memory is released before the next is read
- `break` or `drop(file_stream)` stops all I/O immediately — no wasted reads

#### Example 3: With Partition Pruning (Last N Hours)

Shows how hourly partitioning + sidecar sorting enables sub-second queries on huge tables:

```rust
use deltalake::{DeltaTableBuilder, kernel::Expression};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let table = DeltaTableBuilder::from_uri("abfss://container@account.dfs.core.windows.net/table")
        .with_storage_options(azure_credentials())
        .build()
        .await?;
    
    // Query: last 2 hours of data
    // With sidecar sorted by _event_hour, only 1-2 row groups are read
    let scan = table.scan()
        .with_predicate(
            col("_event_hour").gt_eq(lit("2026021022"))  // 10 PM today
                .and(col("_event_hour").lt_eq(lit("2026021023")))  // 11 PM today
        )
        .build()?;
    
    // For a 10M-file table:
    //   Without sidecar: loads ALL 10M file metadata → 10+ GB, minutes
    //   With sidecar:    reads 1 row group (10K files) → 5 MB, 200ms
    let batches = scan.execute().await?;
    
    for batch in batches {
        process(batch?);
    }
    
    Ok(())
}
```

#### Example 4: Python Usage

The Python bindings expose the same transparent optimization:

```python
import deltalake

# Open table — identical to today
dt = deltalake.DeltaTable(
    "abfss://container@account.dfs.core.windows.net/table",
    storage_options=azure_credentials(),
)

# Option A: Get file URIs as a lazy iterator (for custom processing)
# Internally streams from sidecar with bounded memory
for file_info in dt.file_infos(predicate="_event_hour >= '2026021022'"):
    print(f"File: {file_info.path}, Size: {file_info.size}")
    # Process immediately — don't wait for all files
    
# Option B: Convert to PyArrow dataset (standard path)
# Sidecar optimization happens transparently during file discovery
dataset = dt.to_pyarrow_dataset(
    partitions=[("_event_hour", ">=", "2026021022")],
)
table = dataset.to_table(columns=["device_id", "value", "timestamp"])
print(f"Got {table.num_rows} rows from {len(dataset.get_fragments())} files")

# Option C: Use with pandas (small result sets)
df = dt.to_pandas(
    partitions=[("_event_hour", "=", "2026021023")],
    columns=["device_id", "value"],
)
```

#### Example 5: Kusto External Table Query (End-User Perspective)

From the Kusto query perspective, nothing changes. The optimization is fully transparent:

```kql
// Query 1: Recent data (hits 1 sidecar row group)
external_table('TelemetryDelta')
| where _event_hour >= '2026021022'
| where device_id == 'sensor-42'
| take 100

// Query 2: Full scan with limit (early termination after ~1 row group)
external_table('TelemetryDelta')
| take 100

// Query 3: Aggregation over time window (hits 2-3 row groups)
external_table('TelemetryDelta')
| where _event_hour between ('2026021000' .. '2026021023')
| summarize avg(value) by bin(timestamp, 1h), device_id
```

All three queries benefit from:
- **Streaming**: Kusto receives files as they're discovered, starts parallel reads immediately
- **Bounded memory**: delta-rs never holds more than ~50 MB of metadata
- **Row-group pruning**: Only matching sidecar row groups are read
- **Early termination**: `take 100` stops metadata streaming after enough files

---

## Summary

**Project Sidecar** solves the OOM and blocking problems for ADX/Kusto Delta table queries by:

1. **Writing optimized metadata at ingestion time** — native columnar stats, sorted by partition
2. **Streaming metadata at query time** — bounded memory, parallel log tailing, row-group pruning
3. **Keeping V1 compatibility** — sidecar is additive, legacy readers unaffected
4. **Contributing to Kusto** — improvements flow into the Kusto delta-rs driver

The combination of ingestion-side optimization (sidecar) and reader-side optimization (streaming) reduces metadata memory from **10+ GB to ~5-50 MB** and query start latency from **minutes to sub-second**.

---

## Appendix A: Enhancement Ladder — Performance & Memory at Each Level

Each level is **additive** — you can stop at any rung and get value. Numbers are for a 10M-file table with a "last 1 hour" query.

### The Ladder

| Level | Enhancement | Memory | Time to First File | Time to Complete | Ingestion Change? | Reader Change? |
|-------|------------|--------|--------------------|-----------------|--------------------|----------------|
| **L0** | **Baseline** — V1 checkpoint, eager load | 10-50 GB (OOM) | 60+ s | 60+ s | — | — |
| **L1** | **+ Lazy Iterator** — stream V1 checkpoint row-by-row | ~150 MB | ~5 s | ~60 s | No | ✅ Stream instead of materialize |
| **L2** | **+ Parallel log tailing** — fetch JSONs concurrently | ~150 MB | ~1 s | ~55 s | No | ✅ Concurrent commit fetch |
| **L3** | **+ Sidecar** — native Parquet stats (no JSON parsing) | ~50 MB | ~500 ms | ~30 s | ✅ Write sidecar | ✅ Discover + stream sidecar |
| **L4** | **+ Manifest** — pre-extracted footer, skip Parquet footer dance | ~50 MB | ~200 ms | ~30 s | ✅ Write manifest | ✅ 3-tier fallback |
| **L5** | **+ Column projection** — read only needed stat columns from sidecar | ~20 MB | ~200 ms | ~20 s | No | ✅ Project columns |
| **L6** | **+ Hourly partitioning** — sorted sidecar, row-group pruning | ~5 MB | ~200 ms | **~1 s** | ✅ Time-aligned files | ✅ Row-group pruning |
| **L7** | **+ Early termination** — LIMIT stops streaming | ~5 MB | ~200 ms | **~200 ms** (`take 100`) | No | ✅ LIMIT support |

### Visual: Cumulative Impact

```
Peak Memory (10M files):
  L0  ████████████████████████████████████████████████████  50 GB (OOM)
  L1  ███                                                    150 MB
  L2  ███                                                    150 MB
  L3  ██                                                     50 MB
  L4  ██                                                     50 MB
  L5  █                                                      20 MB
  L6  ▏                                                      5 MB
  L7  ▏                                                      5 MB

Time to First File Yielded:
  L0  ████████████████████████████████████████████████████  60 s
  L1  ████                                                   5 s
  L2  █                                                      1 s
  L3  ▏                                                      500 ms
  L4  ▏                                                      200 ms
  L5  ▏                                                      200 ms
  L6  ▏                                                      200 ms
  L7  ▏                                                      200 ms

Time to Complete ("last 1 hour" query):
  L0  ████████████████████████████████████████████████████  60 s
  L1  ████████████████████████████████████████████████████  60 s
  L2  ███████████████████████████████████████████████        55 s
  L3  ██████████████████████████                             30 s
  L4  ██████████████████████████                             30 s
  L5  █████████████████                                      20 s
  L6  █                                                      1 s
  L7  ▏                                                      200 ms
```

### Key Observations

1. **L1 (Lazy Iterator) is the biggest single memory win** — 50 GB → 150 MB. Prevents OOM by itself.
2. **L3 (Sidecar) is the biggest stats efficiency win** — eliminates JSON parsing, 40x smaller per-file metadata, 3x less data per row group.
3. **L6 (Partitioning) is the biggest query speed win** — for time-bounded queries, reduces metadata read by 99%+. This is where "last 1 hour" goes from 30s to 1s.
4. **L7 (Early termination) is free** — once you have streaming (L1), LIMIT support is automatic. `take 100` completes in ~200ms regardless of table size.
5. **L1 + L2 require zero ingestion changes** — pure reader-side improvements, deployable immediately.
6. **L3-L6 require ingestion changes** — but each is additive on top of the previous. You can deploy L3 without L6, etc.
7. **Each level is independently deployable and testable** — no big-bang release required.

### Mapping to Implementation Phases

| Implementation Phase | Levels Delivered | Key Win |
|---------------------|-----------------|---------|
| Phase 1 (Sidecar Writer) | Prepares L3, L4 | Sidecar + manifest files exist |
| Phase 2 (Sidecar Reader) | L1 + L2 + L3 + L4 + L5 + L7 | Streaming + sidecar + early termination |
| Phase 3 (Hourly Partitioning) | L6 | Row-group pruning for time queries |

### What If You Can Only Do One Thing?

| Constraint | Best single enhancement | Impact |
|-----------|------------------------|--------|
| Can't change ingestion | **L1 (Lazy Iterator)** | OOM → 150 MB. Doesn't speed up queries but prevents crashes. |
| Can change ingestion, want fastest time-to-value | **L1 + L3 (Lazy + Sidecar)** | 50 MB memory, 500ms to first file, 30s full scan |
| Want maximum performance for time queries | **Full stack L1-L7** | 5 MB memory, 200ms to first file, 200ms for `take 100` |

---

## Appendix B: Implementation Phases

### Assumptions

| Factor | Value |
|--------|-------|
| **Team size** | 1 developer |
| **Developer level** | Principal-level, highly experienced |
| **Primary language** | C# (expert) |
| **Secondary languages** | C++ (past), Python, Node.js/TypeScript (past) |
| **Rust experience** | None |
| **delta-rs knowledge** | Low |
| **Ingestion pipeline language** | C# (believed) |
| **Access** | Full — both ingestion and query code |
| **AI tooling** | Anthropic Opus 4.6, no token limits |
| **Testing requirement** | Thorough test suite required |

### Language Domain Split

The project spans two language domains. This is important because the developer's C# expertise can be leveraged for the ingestion side, while Rust is only needed for the reader (delta-rs) side:

| Component | Language | Developer Comfort | Phase |
|-----------|----------|-------------------|-------|
| Sidecar Parquet writer | C# | ✅ Expert | Phase 1 |
| Manifest JSON writer | C# | ✅ Expert | Phase 1 |
| Hour-aligned file splitting | C# | ✅ Expert | Phase 1 |
| Sidecar reader (delta-rs) | Rust | ❌ New | Phase 3 |
| Parallel log tailing (delta-rs) | Rust | ❌ New | Phase 3 |
| Streaming pipeline (delta-rs) | Rust | ❌ New | Phase 3 |
| Kusto connector integration | Rust | ❌ New | Phase 4 |

**Key insight: C# phases first = early value.** Sidecars are produced by week 3. They can be verified independently and are ready for the reader to consume once Phase 3 begins.

### AI Acceleration Factor

With Opus-class AI and no token limits, development velocity is significantly enhanced:

| Task | Without AI | With AI (Opus-level) | Savings |
|------|-----------|---------------------|---------|
| Rust fundamentals (for C# dev) | 5-6 weeks | 3 weeks | ~50% |
| Navigate delta-rs codebase | 2-3 weeks | 1 week | ~60% |
| Write async streaming code | 2 weeks | 1 week | ~50% |
| Write Parquet reader/writer (Rust) | 1-2 weeks | 3-5 days | ~60% |
| Debug borrow checker errors | Hours per error | Minutes per error | ~80% |
| Write comprehensive tests | 30% of dev time | 15% of dev time | ~50% |
| Code review (self) | Manual | AI reviews for correctness + idiomatic Rust | — |

**Overall estimated acceleration: 30-35% total time reduction** compared to solo development without AI.

### Tools & Knowledge Required

| Area | Tools / Skills | Learning Time (AI-assisted) |
|------|---------------|---------------------------|
| C# Parquet writing | Apache.Arrow for .NET or Parquet.NET | 2-3 days (familiar ecosystem) |
| Rust fundamentals | Ownership, borrowing, lifetimes, enums, Result/Option, traits, cargo | 1-2 weeks |
| Async Rust | tokio, async/await, futures::Stream, Pin | 1 week |
| Arrow/Parquet in Rust | `arrow-rs`, `parquet-rs`, RecordBatch, Schema | 3-5 days |
| delta-rs internals | `Snapshot`, `LogStore`, `EagerSnapshot`, `DeltaScan`, `checkpoints.rs` | 1 week |
| Delta Lake protocol | V1 spec, checkpoint format, action types, DVs | 2-3 days |
| Azure Blob Storage | `object_store` crate (Rust), Azure.Storage.Blobs (C#) | 2-3 days |
| Testing | `tokio::test`, mock ObjectStore, xUnit (C#) | Ongoing |

---

### Phase 1: Sidecar + Manifest Writer in C# (Weeks 1-3)

**Language: C# (developer's comfort zone)**  
**Goal**: Ingestion pipeline writes sidecar + manifest alongside V1 checkpoint, with hour-aligned file splitting

| Task | Estimate | Notes |
|------|----------|-------|
| Design sidecar Parquet schema | 2 days | Map from document spec to Arrow.NET schema |
| Implement sidecar Parquet writer | 4 days | Arrow.NET or Parquet.NET. Familiar C# territory. |
| Sort by `_event_hour` before writing | 2 days | LINQ sort + Parquet row group configuration |
| Manifest JSON writer | 1 day | `System.Text.Json` — trivial |
| Hour-aligned file splitting in ingestion | 3 days | Group incoming rows by `_event_hour`, write separate Parquet files |
| Integration with existing C# ingestion pipeline | 3 days | Wire into checkpoint trigger |
| Sidecar + manifest cleanup in VACUUM | 1 day | Delete old sidecars/manifests |
| Tests (unit + integration) | 3 days | C# test frameworks — fast to write |
| Buffer | 2 days | First time writing Parquet with Arrow.NET |

**Deliverable**: Ingestion produces V1 checkpoint + `vN.parquet` sidecar + `vN.manifest.json`. Verified independently with Parquet viewers / Python scripts.

### Phase 2: Rust & delta-rs Ramp-Up (Weeks 4-6)

**Language: Rust (learning phase)**  
**Goal**: Working Rust fluency + delta-rs codebase understanding

| Week | Focus | Deliverable |
|------|-------|-------------|
| Week 4 | Rust fundamentals: ownership, borrowing, lifetimes, enums, pattern matching, error handling (`Result`, `?`), traits, cargo, modules. AI generates exercises tailored to the project domain. | Can write and compile basic Rust programs. Small CLI tool that reads a Parquet file. |
| Week 5 | Async Rust: tokio, async/await, `futures::Stream`, `Pin`. Arrow-rs and parquet-rs crate basics. Write async Parquet reader. | Can write async Parquet row-group reader in Rust. Understands the `Stream` trait. |
| Week 6 | delta-rs deep dive: clone repo, build, run tests. Read `kernel/snapshot`, `logstore`, `protocol/checkpoints.rs`. Understand `EagerSnapshot` load flow. Make a small modification and run tests. | Can navigate delta-rs, understand snapshot loading, run test suite, make targeted changes. |

**AI usage**: Opus generates Rust exercises in terms of C# equivalents ("Rust's `Result<T,E>` is like C#'s `Result` pattern but enforced by the compiler"), walks through delta-rs architecture, creates annotated code tours.

### Phase 3: Sidecar Reader in delta-rs (Weeks 7-13)

**Language: Rust**  
**Goal**: delta-rs discovers and streams sidecar with bounded memory

| Task | Estimate | Notes |
|------|----------|-------|
| 3-tier discovery (manifest → footer → V1) | 4 days | `ObjectStore` GET/HEAD, `serde_json`. First real delta-rs modification. AI-heavy. |
| Parallel JSON commit fetching | 1 week | `futures::join_all`, error handling, version ordering. Tests against real JSON commits. |
| Tombstone + override set construction | 3 days | `HashSet` logic — straightforward algorithm. Rust ownership adds some friction. |
| Sidecar streaming with row-group batching | 2 weeks | **Hardest task.** Async `Stream` + Parquet row group reader + column projection + memory management. AI generates boilerplate. |
| Row-group pruning (manifest-based) | 3 days | JSON manifest → decide which row groups to read. |
| Integration with `Snapshot` / `LogStore` | 1 week | Modifying delta-rs internals. Requires understanding trait hierarchy. |
| DV metadata pass-through | 2 days | Carry DV fields from sidecar and JSON adds to output. |
| V1 checkpoint fallback path | 2 days | Wire existing code path as fallback when no sidecar. |
| Tests (thorough) | 1.5 weeks | Edge cases: tombstones, DV updates, overrides, streaming, early termination, fallback. Generate test Delta tables from C# side. |
| Buffer for Rust learning friction | 3 days | Borrow checker, lifetime issues in async contexts. |

**Total: ~7 weeks.** Would be 4-5 weeks for an experienced Rust developer. Extra 2-3 weeks account for learning friction, substantially reduced by AI.

**Deliverable**: delta-rs can query tables using sidecar with bounded memory, falling back to V1 when sidecar is absent.

### Phase 4: Kusto Integration (Weeks 14-17)

**Language: Rust (delta-rs side)**  
**Goal**: Kusto queries Delta tables using sidecar-optimized path

| Task | Estimate | Notes |
|------|----------|-------|
| Study Kusto delta connector | 1 week | How does Kusto call delta-rs? Where does it get the file list? |
| Wire sidecar-aware path into connector | 1 week | Modify the integration point |
| End-to-end testing with Kusto | 1 week | External table queries against tables with sidecars |
| PR + review process | 1 week | Internal or upstream contribution |

**Total: ~4 weeks** (development time; calendar time may be longer if blocked on reviews)

**Deliverable**: Kusto queries benefit from sidecar optimization.

### Phase 5: Production Readiness (Weeks 18-20)

**Language: Both C# and Rust**  
**Goal**: Hardened, monitored, documented

| Task | Estimate | Notes |
|------|----------|-------|
| Stress testing at scale | 1 week | 10M-file tables, OOM regression tests |
| Monitoring + logging | 3 days | Sidecar hit rate, pruning effectiveness, memory usage |
| Documentation | 3 days | Operations guide, troubleshooting |
| Security review | 2 days | Azure credential handling (both sides) |
| Final polish + edge cases | 3 days | |

**Total: ~3 weeks**

**Deliverable**: Production-ready system with monitoring and documentation.

---

### Timeline Summary

| Phase | Language | Duration | Cumulative | Risk |
|-------|----------|----------|------------|------|
| Phase 1: Sidecar Writer | C# | 3 weeks | 3 weeks | **Low** (comfort zone) |
| Phase 2: Rust Ramp-Up | Rust | 3 weeks | 6 weeks | Medium (AI-accelerated) |
| Phase 3: Sidecar Reader | Rust | 7 weeks | 13 weeks | **High** (hardest phase) |
| Phase 4: Kusto Integration | Rust | 4 weeks | 17 weeks | Medium (external dep) |
| Phase 5: Production | Both | 3 weeks | 20 weeks | Low |
| **Total** | | **20 weeks** | | |

### Range Estimate

| Scenario | Total | Notes |
|----------|-------|-------|
| **Optimistic** | 16 weeks | Rust clicks fast (C++ background helps), AI handles 60%+ of code generation |
| **Expected** | 20 weeks | Normal learning curve with AI assistance |
| **Pessimistic** | 26 weeks | Rust async harder than expected, delta-rs internals resist modification, Kusto team blocks |

### Without AI vs. With AI

| Metric | Without AI | With Opus-class AI |
|--------|-----------|-------------------|
| Rust ramp-up | 5-6 weeks | 3 weeks |
| Phase 3 (reader) | 10-12 weeks | 7 weeks |
| Total project | 26-30 weeks | 20 weeks |
| **Time savings** | — | **~30-35%** |

### Key Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Phase 3 complexity** | Could extend by 2-4 weeks | AI assistance, incremental delivery, test against real C#-produced sidecars |
| **Rust async learning curve** | `Pin<Box<dyn Stream + Send>>` and lifetimes can be time sinks | AI explains and generates patterns; C++ background helps with ownership mental model |
| **delta-rs internal architecture** | May be harder to modify than expected | Start with small PRs, understand existing test suite first |
| **Kusto team dependency** | Phase 4 calendar time ≠ dev time | Start relationship early, share design doc, get early feedback |
| **Testing at scale** | Need 10M-file Delta tables | Generate from C# ingestion pipeline (Phase 1 deliverable) |

---

### AI Capability Analysis: Can Opus 4.6 Do All the Coding and Testing?

**Short answer: Yes for ~80% of the work. No for ~20%.** The developer's role shifts from "writing code" to "directing AI + reviewing + debugging + making architectural decisions."

#### What AI Can Do Well (~80%)

**C# Side (Phase 1) — ~95% AI-generated:**

| Task | AI Capability | Why |
|------|--------------|-----|
| Sidecar Parquet writer (Arrow.NET) | ✅ Excellent | Well-documented APIs, many examples in training data |
| Manifest JSON writer | ✅ Trivial | `System.Text.Json` serialization |
| Hour-aligned file splitting | ✅ Excellent | Standard data partitioning logic |
| Unit tests (xUnit/NUnit) | ✅ Excellent | Can generate comprehensive test cases including edge cases |
| Integration tests | ✅ Good | Can scaffold test harnesses, mock Azure storage |

**Rust Side (Phases 2-3) — ~70% AI-generated:**

| Task | AI Capability | Why |
|------|--------------|-----|
| Rust fundamentals exercises | ✅ Excellent | Can teach Rust by analogy to C# |
| Parquet reading with `parquet-rs` | ✅ Good | Known APIs, can generate correct code |
| `serde_json` deserialization | ✅ Excellent | Standard Rust |
| `HashSet`-based tombstone logic | ✅ Excellent | Pure algorithm, no framework complexity |
| Unit tests (`#[tokio::test]`) | ✅ Good | Can generate test structure and assertions |
| Async streaming (`futures::Stream`) | ⚠️ Moderate | Can generate patterns but may need iteration on lifetimes |
| `Pin<Box<dyn Stream>>` patterns | ⚠️ Moderate | Common source of compile errors; AI can fix but may take multiple rounds |
| ObjectStore mock for tests | ⚠️ Moderate | Less training data on mock ObjectStore patterns |

**delta-rs Modification (Phase 3 core) — ~50% AI-generated:**

| Task | AI Capability | Why |
|------|--------------|-----|
| Understanding `EagerSnapshot` load flow | ⚠️ Moderate | AI can read the code and explain it, but delta-rs has complex internal architecture |
| Modifying `Snapshot` trait implementation | ⚠️ Moderate | Requires understanding the full trait hierarchy and downstream consumers |
| Integration with `LogStore` | ❌ Limited | delta-rs internals change between versions; AI may generate code that doesn't fit current architecture |
| Integration tests against real delta-rs | ⚠️ Moderate | Can scaffold but may miss subtle correctness issues |

#### What AI Cannot Do (~20%)

**1. Architectural decisions in unfamiliar codebases**

AI can read delta-rs code and explain what it does, but it cannot reliably determine the *right place* to make modifications. The developer must understand:
- Which `Snapshot` variant to extend vs. replace
- How `DeltaScan` receives its file list and where to inject streaming
- Whether to modify `EagerSnapshot` or create a new `LazySnapshot`

*Mitigation:* Use AI to *explore* (read files, explain architecture, trace call paths) but make architectural decisions yourself.

**2. Debugging async lifetime errors in context**

When you get a 50-line Rust compiler error about lifetimes in an async block referencing a borrowed `ObjectStore`, AI can suggest fixes — but they may not compile either. This can turn into a back-and-forth:

```
Developer: "Fix this lifetime error"
AI: *generates fix*
Compiler: "error[E0597]: `store` does not live long enough"
Developer: "That didn't work"
AI: *generates different fix*
Compiler: "error[E0521]: borrowed data escapes outside of async block"
... (3-5 iterations before converging)
```

*Mitigation:* The C++ background helps. A developer who understands *why* ownership matters will converge faster than one who just tries suggestions blindly.

**3. Testing correctness at the semantic level**

AI can generate test cases for known patterns, but it may miss:
- Subtle ordering issues in concurrent JSON fetches
- Race conditions in streaming with early termination
- Edge cases where DV updates span multiple commits
- Sidecar/checkpoint version mismatch scenarios

*Mitigation:* The developer (principal level) needs to think through correctness invariants and tell the AI which edge cases to test. Don't rely on AI to *discover* edge cases — it's good at *implementing* tests you describe.

**4. Production integration and debugging**

When something fails at scale (10M files on Azure Blob), the AI can't:
- Observe runtime behavior
- Interpret memory profiling output
- Debug network-specific issues (throttling, timeouts)
- Understand Kusto-specific integration quirks

*Mitigation:* Standard engineering debugging. AI helps with "what could cause this error?" and "generate a diagnostic tool."

#### Recommended Workflow: AI as Coder, Human as Architect

```
Developer's role:                          AI's role:
─────────────────                          ─────────
1. Define WHAT to build                    1. Generate ~80% of production code
   (architecture, module boundaries)       2. Generate ~90% of test scaffolding
2. Tell AI to generate each component      3. Explain unfamiliar code (delta-rs)
3. Review generated code for correctness   4. Fix compiler errors (borrow checker)
4. Run tests, fix failures (with AI help)  5. Suggest API patterns (arrow-rs, etc.)
5. Make architectural decisions when        6. Review code for idiomatic Rust
   AI-generated code doesn't fit           7. Generate documentation
6. Design test scenarios (edge cases)
7. Debug integration issues
```

#### Impact on Timeline

| AI code generation accuracy | Timeline impact |
|----------------------------|----------------|
| ~90% correct (optimistic) | **16-17 weeks** |
| ~80% correct (expected) | **20 weeks** |
| ~60% correct (pessimistic — lots of debugging AI output) | **24-26 weeks** |

The key variable is not whether AI can *generate* the code, but whether the generated code *fits correctly into delta-rs's architecture*. For greenfield code (sidecar writer in C#, standalone Rust modules), AI is nearly 100% effective. For modifying an existing complex Rust codebase (delta-rs internals), effectiveness drops to 50-60%.

#### Bottom Line

**Opus 4.6 can write the code. The developer's job shifts from "writing code" to "directing the AI + reviewing + debugging + making architectural decisions."** This is a different skill set — closer to a senior architect reviewing junior engineers' PRs — which aligns well with a principal-level developer's strengths.

---

## Appendix C: Beyond L7 — Further Optimizations & Edge Cases

### Where You Stand After L7

With the full sidecar stack (L0→L7), for a "last 1 hour" query on a 10M-file table:

| Metric | Value | Was |
|--------|-------|-----|
| Memory | ~5 MB | 50 GB |
| Time to first file | ~200 ms | 60 s |
| Time to complete (time-bounded) | ~200 ms – 1 s | 60 s |

**The metadata layer is essentially solved.** The remaining bottleneck is **data reading** (actual Parquet files), not metadata discovery. Further metadata optimization yields diminishing returns — you're already at:
- 1 HTTP request for manifest (~30 ms)
- 1 row group read (~50 ms)
- 10 tiny JSON fetches in parallel (~100 ms)

### Is It Worth Optimizing More?

**For the common case: No.** L7 is the sweet spot. But there are specific scenarios where additional optimization IS worth it.

---

### Scenario 1: Cold Start / First Query After Deploy (HIGH VALUE)

**When it happens**: Kusto node starts, first query hits a Delta table. No cache, no warm state.

**The problem**: The first query pays the full discovery cost (~200 ms). Subsequent queries for the same table version re-discover everything.

**Optimization (L8): Sidecar metadata caching**

Cache the manifest + sidecar footer in memory across queries. Second query for the same table version is near-instant (~1 ms).

```rust
struct SidecarCache {
    // LRU cache: table_uri → (version, manifest, row_group_index)
    cache: LruCache<String, (i64, Manifest, Vec<RowGroupMeta>)>,
}
```

Cache invalidation is simple: if `_last_checkpoint` version > cached version → evict and reload.

**Worth it?** **Yes** — if the same Kusto node queries the same table repeatedly (which it almost certainly does). High value, low effort.

**Effort: 3-5 days.**

---

### Scenario 2: Massive Checkpoint Gap (100+ JSON Commits)

**When it happens**: Checkpoints are configured infrequently (e.g., every 100 commits instead of every 10), or ingestion bursts cause many commits between checkpoints.

**The problem**: With 100 JSONs to fetch, even in parallel that's ~500 ms-1 s (Azure Blob throttling, connection limits). The tombstone set could also grow large.

**Optimization (L8-alt): Incremental delta sidecars**

Instead of individual JSON commits, write **delta sidecar files** that batch multiple commits:

```
_sidecars/v1000.parquet            ← Full sidecar at checkpoint 1000
_sidecars/v1000_1010.delta.parquet ← Adds/removes from commits 1001-1010
_sidecars/v1000_1020.delta.parquet ← Adds/removes from commits 1011-1020
```

The reader reads the delta files (binary Parquet, fast) instead of 100 individual JSON commits (text, slow to parse).

**Worth it?** Only if you regularly have 50+ commits between checkpoints. If you checkpoint every 10 commits, this adds complexity for negligible gain.

**Effort: 2-3 weeks.**

---

### Scenario 3: Very Wide Tables (500+ Columns)

**When it happens**: IoT/telemetry tables with hundreds of sensors, each as a separate column.

**The problem**: Even with column projection, a 1000-column table has 3000 stat columns in the sidecar (`min/max/null_count` × 1000). Row groups are larger, and column projection still has overhead proportional to the number of Parquet columns in the footer metadata.

**Optimization: Tiered stats (two-file sidecar)**

Split the sidecar into two files:

| File | Contents | Size per file entry | When loaded |
|------|----------|--------------------|----|
| `v1000.index.parquet` | path, size, partition key, num_records | ~50 bytes | Always |
| `v1000.stats.parquet` | Full column min/max/null_count | ~200+ bytes | Only for predicate pushdown |

Most queries only need the index file. Stats are loaded on-demand for specific columns referenced in the predicate.

**Worth it?** Only if tables have 500+ columns AND predicate pushdown is used on many columns. For 50-100 column tables, single sidecar with projection is sufficient.

**Effort: 1-2 weeks.**

---

### Scenario 4: OPTIMIZE Creates Massive Tombstone Sets

**When it happens**: You run OPTIMIZE (compaction) on a large partition — removes thousands of small files and replaces them with fewer large files.

**The problem**: A single OPTIMIZE commit can have 50K+ removes. The `HashSet<String>` tombstone set uses 50K × ~100 bytes = ~5 MB. Not catastrophic, but significant.

**Optimization: Bloom filter for tombstone set**

Replace `HashSet<String>` with a Bloom filter (~0.1% false positive rate):
- 50K entries × 10 bits ≈ 60 KB instead of 5 MB
- False positives cause a file to be skipped (rare, acceptable for point-in-time queries)

**Worth it?** Only after large OPTIMIZE operations. For append-only workloads (your primary case), tombstones are empty and this is irrelevant.

**Effort: 2-3 days.** Bloom filter crates exist in Rust (`bloomfilter`, `probabilistic-collections`).

⚠️ **Caution**: Bloom filter false positives mean a valid file could be skipped. This is ONLY acceptable if the false positive rate is extremely low (~0.01%) or if the downstream query can tolerate very rare missing rows. For financial/compliance data, stick with the exact `HashSet`.

---

### Scenario 5: Multi-Table Joins in Kusto

**When it happens**: Kusto query joins two or more external Delta tables.

**The problem**: Each table independently loads its metadata and streams files. If both are large, memory is 2× the single-table cost, and both compete for network bandwidth.

**Optimization: Shared ObjectStore connection pool + coordinated streaming**

Ensure both sidecar streams share a connection pool to Azure Blob and don't saturate the network with concurrent requests.

**Worth it?** Only if multi-table joins are common and both tables are very large. In practice, Kusto likely handles connection management at the engine level already.

**Effort: 1 week.**

---

### Edge Cases You MUST Test

These are not optimizations — they are **correctness requirements** that must be covered in Phase 3 testing.

#### Edge Case 1: Stale or Missing Sidecar

**Scenario**: Ingestion pipeline crashes between writing V1 checkpoint and writing the sidecar. Or: sidecar is written but manifest isn't.

| State | Expected Behavior |
|-------|------------------|
| Manifest exists, sidecar exists, versions match | ✅ Happy path — use sidecar |
| Manifest 404, sidecar exists | ✅ Tier 2 — read sidecar via Parquet footer |
| Manifest 404, sidecar 404 | ✅ Tier 3 — fall back to V1 checkpoint |
| Manifest exists, sidecar 404 (corrupted) | ⚠️ Must detect and fall back to V1 |
| Manifest exists but points to wrong sidecar file | ⚠️ Must validate and fall back to V1 |
| Sidecar exists for version 990 but checkpoint is 1000 | ⚠️ Version mismatch — must fall back to V1 |

**Test plan**: Create each of these states on Azure Blob and verify fallback behavior.

#### Edge Case 2: Concurrent Checkpointing

**Scenario**: Two ingestion instances trigger a checkpoint at the same time for the same version. Both write sidecars concurrently.

**Risk**: One instance partially writes the sidecar, the other overwrites it. The reader could see a truncated or corrupted Parquet file.

**Mitigation**:
1. The reader must handle Parquet parse failures gracefully → fall back to V1
2. Consider write-to-temp-then-rename pattern for atomic sidecar writes (Azure Blob supports this via copy+delete)

**Test plan**: Simulate concurrent writes, verify reader handles corruption.

#### Edge Case 3: Schema Evolution Between Checkpoints

**Scenario**: Commit C+5 adds a new column. The sidecar (at version C) doesn't have stats for this column. Commits C+5..V have `add` actions with stats for the new column.

**Expected behavior**:
- JSON adds (C+5..V): have stats for new column → engine can use them
- Sidecar files (≤C): missing stats for new column → engine must handle null/missing stats gracefully
- Next checkpoint: new sidecar will include the new column

**This is not a bug** — it's expected behavior. But the reader must not crash when sidecar schema differs from current table schema.

#### Edge Case 4: DV Update Spanning Multiple Commits

**Scenario**:
- Commit C+3: remove(file_A), add(file_A, DV_v2) — 5 deleted rows
- Commit C+7: remove(file_A), add(file_A, DV_v3) — 8 deleted rows

The file is removed and re-added TWICE. The override set must track the LATEST version.

**Expected behavior**: The second `add(file_A, DV_v3)` from commit C+7 is what gets yielded. The first re-add from C+3 is itself overridden.

**Test plan**: Create multi-commit DV updates for the same file, verify only the latest DV is yielded.

#### Edge Case 5: Empty Sidecar (No Files at Checkpoint)

**Scenario**: A table is created, all files are removed by OPTIMIZE, then new files are added. At checkpoint time, the table has 0 active files.

**Expected behavior**: Sidecar is a valid Parquet file with 0 rows. Reader streams 0 files from sidecar, yields only JSON adds.

#### Edge Case 6: Very Large Single Commit (Bulk Import)

**Scenario**: A single commit adds 1M files (bulk import or large OPTIMIZE output).

**Expected behavior**: The parallel JSON fetch retrieves one very large JSON file. Parsing 1M add actions from a single JSON should not OOM — process them in a streaming fashion or accept the single-commit memory cost (typically manageable since it's one commit).

**Test plan**: Generate a commit with 100K+ adds, verify memory stays bounded during processing.

---

### Recommendation Matrix

| Optimization | Priority | When to Implement | Effort |
|-------------|----------|------------------|--------|
| **Sidecar metadata caching** | **🟢 High** | After Phase 3 | 3-5 days |
| **Stale sidecar edge cases** | **🔴 Must have** | Phase 3 testing | 3-5 days |
| **Concurrent checkpoint handling** | **🔴 Must have** | Phase 3 testing | 2-3 days |
| **Schema evolution handling** | **🔴 Must have** | Phase 3 testing | 2 days |
| **Multi-commit DV updates** | **🔴 Must have** | Phase 3 testing | 2 days |
| Incremental delta sidecars | 🟡 Low | Only if checkpoint gap > 50 | 2-3 weeks |
| Tiered stats (wide tables) | 🟡 Low | Only if 500+ columns | 1-2 weeks |
| Bloom filter for tombstones | 🟡 Low | Only if OPTIMIZE creates 50K+ removes | 2-3 days |
| Multi-table coordination | ⚪ Very Low | Only if common join pattern | 1 week |

**Bottom line**: The design as-is (L0-L7) is the right stopping point for the initial implementation. Add **caching** (3-5 days) after Phase 3 as a quick win. Focus testing energy on the **must-have edge cases** — those are the ones that will bite you in production.
