# Test Framework Plan

* Project Drasi — April 29, 2026 — Ruokun Niu (@ruokun-niu)

## Test Framework Requirements


**Runnable manually.** Developers must be able to run the full test suite from their local machine. The existing building_comfort examples on the `feature-lib` branch demonstrate three distinct run patterns depending on the target:

- **Embedded (drasi-lib in-process):** A single shell script runs the test-service with an embedded Drasi engine — no external processes needed. The script invokes `cargo run --release --manifest-path ./test-service/Cargo.toml -- --config <config.json>` from the `e2e-test-framework` directory. Everything runs in one process.
- **Drasi Server (standalone):** A shell script first builds and starts the drasi-server binary (from a sibling `../../drasi-server` directory) with a `server-config.yaml`, waits for its health check, and then starts the test-service with a separate `config.json` that dispatches changes via HTTP or gRPC to the running server. Two processes run side by side; a `stop.sh` script cleans up both.
- **Drasi Platform (Kubernetes):** The test-service is deployed as a Kubernetes pod alongside Drasi Platform. Tests are controlled via REST API calls (using `curl` or `.http` files) to the test-service's API endpoint, which is port-forwarded from the cluster. The drasi_platform example has sub-variants for different query container index backends (`query_container_default`, `query_container_memory`, `query_container_redis`, `query_container_rocks`).

Each example provides both `run_debug.sh` and `run_release.sh` scripts. For CI, the same scripts (or equivalent `workflow_dispatch` triggers) can be used in GitHub Actions.

**Automatable in CI workflows.** The test suite must run automatically in GitHub Actions. Trigger conditions are TBD — candidates include on every PR, on pushes to `main`/`feature-lib`, or via `workflow_dispatch` for manual CI runs.

#### Current Config Structure

Today, each test target has its own completely separate config file with the test logic (data model, queries, etc.) duplicated inside it. The examples below are from the existing building comfort E2E test on the `feature-lib` branch.

**Embedded config** (`drasi_embedded/config.yaml`) — everything in one file, one process:

```yaml
data_store:
  data_store_path: examples/building_comfort/drasi_embedded/test_data_cache
  delete_on_start: true
  delete_on_stop: true
  test_repos:
    - id: local_dev_repo
      kind: LocalStorage
      source_path: examples/building_comfort/drasi_embedded/dev_repo
      local_tests:
        - test_id: building_comfort
          test_folder: building_comfort

          # Embedded Drasi engine definition
          drasi_servers:
            - id: embedded-drasi-server
              config:
                storage:
                  type: memory
                sources:
                  - id: facilities-db
                    source_type: application
                    auto_start: true
                queries:
                  - id: all-rooms
                    query: "MATCH (r:Room) RETURN elementId(r) AS RoomId, r.temperature, r.humidity, r.co2"
                    sources: [facilities-db]
                    auto_start: true
                reactions:
                  - id: building-comfort-alerts
                    reaction_type: application
                    queries: [all-rooms]
                    auto_start: true

          # ETF source — dispatches via in-process channel
          sources:
            - test_source_id: facilities-db
              kind: Model
              source_change_dispatchers:
                - kind: DrasiServerChannel          # <-- target-specific
                  drasi_server_id: embedded-drasi-server
                  source_id: facilities-db
                  buffer_size: 2048
              model_data_generator:
                kind: BuildingHierarchy
                seed: 123456789
                change_count: 100000
                # ... sensor configs ...

          # ETF reaction observer — receives via in-process channel
          reactions:
            - test_reaction_id: building-comfort
              output_handler:
                kind: DrasiServerChannel            # <-- target-specific
                drasi_server_id: embedded-drasi-server
                reaction_id: building-comfort-alerts
              stop_triggers:
                - kind: RecordCount
                  record_count: 90000

test_run_host:
  test_runs:
    - test_id: building_comfort
      test_repo_id: local_dev_repo
      test_run_id: test_run_001
      drasi_servers:
        - test_drasi_server_id: embedded-drasi-server
          start_immediately: true
      sources:
        - test_source_id: facilities-db
          start_mode: auto
      reactions:
        - test_reaction_id: building-comfort
          start_immediately: true
          output_loggers:
            - kind: PerformanceMetrics
```

**Drasi Server config** (`drasi_server/drasi_server_grpc/`) — two separate files, two processes:

The Drasi Server itself gets `server-config.yaml`:
```yaml
server:
  host: 0.0.0.0
  port: 8080
  disable_persistence: true
sources:
  - id: facilities-db
    source_type: grpc
    auto_start: true
    properties:
      host: 0.0.0.0
      port: 50051
queries:
  - id: building-comfort
    query: "MATCH (r:Room) RETURN r"
    sources: [facilities-db]
    auto_start: true
reactions:
  - id: rooms-grpc
    reaction_type: grpc
    queries: [building-comfort]
    auto_start: true
    properties:
      endpoint: http://127.0.0.1:50052
```

The ETF gets `config.json` with gRPC dispatchers/handlers:
```json
{
  "sources": [{
    "test_source_id": "facilities-db",
    "kind": "Model",
    "source_change_dispatchers": [{
      "kind": "Grpc",
      "host": "localhost",
      "port": 50051,
      "source_id": "facilities-db"
    }],
    "model_data_generator": {
      "kind": "BuildingHierarchy",
      "seed": 123456789,
      "change_count": 100000
    }
  }],
  "reactions": [{
    "test_reaction_id": "building-comfort",
    "output_handler": {
      "kind": "Grpc",
      "host": "0.0.0.0",
      "port": 50052,
      "query_ids": ["building-comfort"]
    },
    "stop_triggers": [{ "kind": "RecordCount", "record_count": 100000 }]
  }]
}
```

#### Issue with existing solution

The core test logic — the BuildingHierarchy model config (seed, change_count, sensor definitions), the query text, and the stop trigger count — is duplicated across every target variant. Adding a new test scenario or modifying an existing one requires updating 3-4 files. The `source_change_dispatchers`, `output_handler`, and `drasi_servers` blocks are the only parts that differ between targets.

#### Proposed Changes

The goal is to make it easy to run the same test against different versions and configurations of Drasi. Within a target mode (embedded, standalone server, platform), the ETF config — model generator, dispatchers, handlers, stop triggers — stays fixed. What changes between variants is only the Drasi engine configuration (storage backend, persistence settings, etc.). So the approach is to keep the ETF config as a self-contained file per target mode, and make the Drasi engine config a separate, swappable file.

Each test has one ETF config per target mode and multiple Drasi config files — one per variant:

```
building_comfort/
├── embedded/
│   ├── config.yaml                  # ETF config (fixed for embedded mode)
│   ├── drasi-memory.yaml            # Drasi engine: storage.type = memory
│   └── drasi-rocksdb.yaml           # Drasi engine: storage.type = file (RocksDB)
├── server-grpc/
│   ├── config.yaml                  # ETF config (fixed for gRPC mode)
│   ├── server-config-memory.yaml    # Drasi server: persistIndex = false
│   └── server-config-rocksdb.yaml   # Drasi server: persistIndex = true
├── server-http/
│   ├── config.yaml                  # ETF config (fixed for HTTP mode)
│   ├── server-config-memory.yaml
│   └── server-config-rocksdb.yaml
└── data/expected/                   # Golden files for result verification
```

**For embedded mode**, the ETF config references the Drasi engine config by file path instead of inlining it. To test a different storage backend, you just swap the filename:

```yaml
# config.yaml (embedded mode — ETF config)
data_store:
  data_store_path: test_data_cache
  delete_on_start: true
  delete_on_stop: true
  test_repos:
    - id: local_dev_repo
      kind: LocalStorage
      source_path: dev_repo
      local_tests:
        - test_id: building_comfort
          test_folder: building_comfort
          drasi_servers:
            - id: embedded-drasi-server
              config_file: drasi-memory.yaml    # <-- swap this to change variant
          sources:
            - test_source_id: facilities-db
              kind: Model
              source_change_dispatchers:
                - kind: DrasiServerChannel
                  drasi_server_id: embedded-drasi-server
                  source_id: facilities-db
                  buffer_size: 2048
              model_data_generator:
                kind: BuildingHierarchy
                seed: 123456789
                change_count: 100000
                # ... full sensor configs ...
          reactions:
            - test_reaction_id: building-comfort
              output_handler:
                kind: DrasiServerChannel
                drasi_server_id: embedded-drasi-server
                reaction_id: building-comfort-alerts
              stop_triggers:
                - kind: RecordCount
                  record_count: 90000

test_run_host:
  test_runs:
    - test_id: building_comfort
      test_repo_id: local_dev_repo
      test_run_id: test_run_001
      drasi_servers:
        - test_drasi_server_id: embedded-drasi-server
          start_immediately: true
      sources:
        - test_source_id: facilities-db
          start_mode: auto
      reactions:
        - test_reaction_id: building-comfort
          start_immediately: true
          output_loggers:
            - kind: PerformanceMetrics
```

The swappable Drasi config files contain only the engine definition:

```yaml
# drasi-memory.yaml
storage:
  type: memory
sources:
  - id: facilities-db
    source_type: application
    auto_start: true
queries:
  - id: all-rooms
    query: "MATCH (r:Room) RETURN elementId(r) AS RoomId, r.temperature, r.humidity, r.co2"
    sources: [facilities-db]
    auto_start: true
reactions:
  - id: building-comfort-alerts
    reaction_type: application
    queries: [all-rooms]
    auto_start: true
```

```yaml
# drasi-rocksdb.yaml — only storage differs
storage:
  type: file
  path: /tmp/drasi-test-index
  persist: true
sources:
  - id: facilities-db
    source_type: application
    auto_start: true
queries:
  - id: all-rooms
    query: "MATCH (r:Room) RETURN elementId(r) AS RoomId, r.temperature, r.humidity, r.co2"
    sources: [facilities-db]
    auto_start: true
reactions:
  - id: building-comfort-alerts
    reaction_type: application
    queries: [all-rooms]
    auto_start: true
```

**For standalone drasi-server mode**, nothing changes in the ETF config. The run script just passes a different `server-config.yaml` to the `drasi-server` binary:

```bash
# run_release.sh — swap the server config to change variant
cargo run --release -p drasi-server -- --config "$SCRIPT_DIR/server-config-rocksdb.yaml"
```

**For drasi-platform mode**, the same principle applies — the ETF deployment config stays the same, and you swap the QueryContainer resource definition to change the storage profile (memory, Redis, RocksDB). This is already how the `drasi_platform/` subdirectories work today (`query_container_default/`, `query_container_memory/`, `query_container_redis/`, `query_container_rocks/`).

**ETF change required:** Add support for `config_file:` as an alternative to inline `config:` in the `drasi_servers` block. When `config_file` is present, the ETF loads the Drasi engine config from the referenced file path (relative to the ETF config file) instead of expecting it inline. This is a small, targeted change — the loaded content has the exact same schema as the existing inline `config:` block.

---

## How Each Target Is Consumed

**drasi-lib (Embedded Mode).** The ETF consumes drasi-lib via the `drasi-core` git submodule in the `test-infra` repo. The `test-run-host` crate depends on it as a Cargo path dependency, so drasi-lib is compiled directly into the test-service binary. Tests use `DrasiServerChannel` dispatchers for zero-network in-process communication. To test a different version, update the submodule pointer. This mode already works today on the `feature-lib` branch.

**drasi-server (Standalone Mode).** The run script downloads a pre-built drasi-server binary from GitHub Releases or the published Docker image, starts it with a `server-config.yaml`, and then runs the ETF test-service separately. The ETF dispatches changes to the server via HTTP or gRPC. For local development, a `--binary` flag can override with a locally-built binary. To test different index backends, swap the server config file.

**drasi-platform (Kubernetes Mode).** The test-service is deployed as a pod alongside Drasi Platform on a Kind/K3D cluster. Communication happens via Dapr or Redis streams. Tests are controlled via the ETF's REST API. To test different index backends, swap the QueryContainer resource definition.



**Public.** All test configuration and test data must be publicly accessible without authentication tokens. The ETF's `GitHub` test repo backend fetches files via the GitHub REST API at runtime, which is subject to rate limiting (60 requests/hour unauthenticated, 5,000 with a PAT). This makes it unsuitable for hosting test data that gets pulled repeatedly across test runs and CI jobs. The options are:

- **Local storage in the repo (recommended for configs and small data):** Test configs, Drasi engine configs, and small golden files live directly in the `test-infra` repo. After cloning (or checking out in CI), the ETF uses the `LocalStorage` backend with a relative filesystem path — no API calls, no rate limits. This works well for YAML configs and small expected-result files. Smaller synthetic datasets (generated by model sources with fixed seeds) also fit here since they are produced at runtime and don't need to be stored.

- **Hugging Face Hub (recommended for test data files):** Public datasets on Hugging Face are freely downloadable without authentication and without the rate limiting issues of GitHub's API. Files are accessible via a predictable HTTP URL pattern: `https://huggingface.co/datasets/{org}/{repo}/resolve/main/{path}`. We would create a public dataset repo (e.g., `drasi-project/test-data`) and push JSONL files to it. HF Hub is git-based and automatically uses Git LFS for large files, so versioning is built in. The ETF would download files with a simple HTTP GET to the resolve URL — no special client library needed. This is free, public, requires no infrastructure to maintain, and has no rate limits for normal usage.

The recommended approach is: **test configs and golden files in the repo** (LocalStorage), **JSONL test data files on Hugging Face Hub** (downloaded via HTTP at test time or pre-cached locally). The ETF would need a small addition to support fetching data from an HTTP URL — either a new `HuggingFace` backend or a generic `HttpDownload` backend that accepts a base URL and file paths. Alternatively, a shell script in the test setup phase could pre-download the data into the local data store path before the ETF starts.

---

## Defined Test Suite

We need a defined test suite for drasi-platform, drasi-server, and drasi-lib that covers four dimensions: component configurations, query complexities, index configurations, and failure recovery.

### Component Configurations

The test suite should exercise a variety of source, query, and reaction configurations. The scenarios to cover include:

- Different source types (model-generated synthetic data, script-replayed recorded data)
- Multi-source queries with synthetic joins
- Fan-out topologies (one source feeding multiple queries)

It is worth noting that drasi-platform, drasi-server, and drasi-lib have different available sources and reactions. For example, drasi-lib and drasi-server can use the `scriptfile` bootstrap provider to load initial data directly from local JSONL files, but the Kubernetes-based drasi-platform cannot access local files — it requires the test infrastructure to bootstrap data by streaming it through the ETF's source dispatchers.

### Query Indexes

Each target supports a different set of query index backends. The test suite must cover all supported configurations per target:

| Target | Available Index Backends |
|--------|------------------------|
| **drasi-platform** | Memory, Redis/Garnet, RocksDB |
| **drasi-server** | Memory (default), RocksDB |
| **drasi-lib** | Memory (default), RocksDB, Redis/Garnet |


### Failure Recovery

The following scenarios are feasible with the current ETF capabilities or with modest extensions:

- **Source pause and resume.** The ETF already supports `pause`/`start` control on sources via its REST API. A test can dispatch N events, pause the source, verify partial results, resume, and verify the final result matches the full run. This validates that no events are lost during a pause/resume cycle.

- **Query stop and restart.** The ETF supports `stop`/`start` on query observers. For embedded mode, the Drasi engine can also stop and restart a query. A test can process events, stop the query, continue dispatching changes (which the query misses), restart the query (triggering re-bootstrap), and verify the query reaches the correct final state.

- **Server process restart (drasi-server).** In standalone mode, the run script can kill and restart the drasi-server binary between test phases. With `persistIndex: false`, the server must re-bootstrap from the source and produce the same results. With `persistIndex: true`, the server should recover from the persisted RocksDB index without re-bootstrapping. This requires the run script to orchestrate pause → kill → restart → resume, which is scriptable but not yet automated in the ETF.

- **Server process restart with state store (drasi-server).** When a source is configured with a `stateStore` (e.g., `kind: redb`), restarting the server should cause the source to resume from where it left off rather than replaying from the beginning. The ETF can verify this by checking that no duplicate reaction outputs are produced after restart.

- **Reaction stop and restart.** Stop a reaction observer while the query continues producing results, then restart it. The reaction should catch up on any results it missed during the gap. The ETF's reaction control APIs (`stop`/`start`) already support this.

The following scenarios require more significant work or depend on features not yet implemented:

- **Checkpoint-based source replay.** The Source Checkpoints design (documented in `drasi-lib/Source-Checkpoints/`) is not yet fully implemented. Once available, tests can verify that a source restarts from its last checkpoint rather than replaying all events.

### Tracing and Metrics Integration

The ETF supports OtelMetric and OtelTrace loggers that can export telemetry to OpenTelemetry-compatible backends. However, drasi-server and drasi-lib do not yet have structured tracing or metrics instrumentation implemented. A separate design document for adding tracing, logging, and metrics integration to drasi-lib is currently under review at [drasi-project/design-documents#7](https://github.com/drasi-project/design-documents/pull/7). We will revisit this section once that design is reviewed and approved, as it will determine what telemetry signals are available for the test framework to consume and verify.

### Result Reporting

After a test run completes, we need to extract key performance and correctness results in a structured, comparable format. The ETF already provides several logger types that capture data during a run. The question is how to consolidate them into actionable test reports.

#### What the ETF captures today

The ETF's logger system writes output per query and per reaction during a test run:

- **PerformanceMetrics logger** (reaction output) — writes a JSON summary at the end of the run:
  ```json
  {
    "start_time_ns": 1627849200000000000,
    "end_time_ns": 1627849260000000000,
    "duration_ns": 60000000000,
    "record_count": 150000,
    "records_per_second": 2500.0,
    "test_run_reaction_id": "repo.test.run001.reaction",
    "timestamp": "2025-07-31T19:45:00Z"
  }
  ```

- **Profiler logger** (query results) — generates detailed profiling data including bootstrap and change processing stats, min/max/avg latencies, and optional visualization images. Outputs to JSONL files with configurable `write_bootstrap_log`, `write_change_log`, and `write_change_image` flags.

- **JsonlFile logger** — writes every query result or reaction output as a JSONL record. This is the raw data needed for correctness verification (comparing against golden files).

- **OtelMetric / OtelTrace loggers** — export to OpenTelemetry-compatible backends (e.g., Prometheus, Jaeger). Useful for live dashboards but not for automated test reporting.

#### Key metrics to report

For each test run, the report should capture:

| Metric | Source | Purpose |
|--------|--------|---------|
| **Bootstrap duration** | Profiler logger | Time to load initial data |
| **Bootstrap throughput** (events/sec) | Profiler logger | Bootstrap ingestion rate |
| **Change processing throughput** (events/sec) | PerformanceMetrics / Profiler | Steady-state throughput |
| **Change processing latency** (min/avg/p95/p99/max) | Profiler logger | Per-event latency distribution |
| **Total events processed** | PerformanceMetrics | Completeness check |
| **Total test duration** | PerformanceMetrics | Wall-clock time |
| **Correctness: pass/fail** | JsonlFile vs golden files | Whether results match expected output |
| **Error count** | Logs / reaction output | Any errors during the run |

#### Proposed reporting approach

1. **Structured summary file.** After each test run, the ETF (or a post-processing script) should produce a single `test-report.json` that consolidates the key metrics from all loggers into one file. This makes it easy to compare runs, track regressions, and display in CI summaries.

   ```json
   {
     "test_id": "building_comfort",
     "target": "embedded-memory",
     "timestamp": "2026-04-29T15:30:00Z",
     "status": "pass",
     "bootstrap": {
       "duration_ms": 75,
       "events": 4210,
       "events_per_sec": 56133
     },
     "changes": {
       "duration_ms": 63,
       "events": 100000,
       "events_per_sec": 15873,
       "latency_ms": {
         "min": 0,
         "avg": 0.063,
         "p95": 0.1,
         "p99": 0.5,
         "max": 1
       }
     },
     "correctness": {
       "expected_records": 90000,
       "actual_records": 90000,
       "mismatches": 0
     }
   }
   ```

2. **Correctness verification.** A post-run step compares the JsonlFile output against golden files. For model-generated tests with fixed seeds, the output should be deterministic — any difference is a failure. The comparison tool should report which records differ (added, missing, or changed fields) to make debugging easier.

3. **CI integration.** In GitHub Actions, the recommended approach is to use `$GITHUB_STEP_SUMMARY` to render a Markdown results table directly on the workflow run's Summary tab. A post-run step parses `test-report.json` and writes key metrics (status, throughput, latency, correctness) into the summary. This is visible immediately without downloading artifacts. The raw `test-report.json` and logger output files should also be uploaded as build artifacts for historical tracking and debugging.

4. **Regression detection.** For performance metrics, we can define thresholds (e.g., throughput must not drop below 80% of baseline, p99 latency must not exceed 2x baseline). The CI step compares the current run's `test-report.json` against a stored baseline and fails if thresholds are breached. Baselines are updated explicitly (not automatically) to avoid ratcheting.

---

## Additional Notes

### Replace ETF Script Bootstrap with the Drasi `scriptfile` Bootstrap Provider

Today, when a test uses recorded data (as opposed to model-generated data), the ETF handles bootstrapping itself: the `Script` kind source has a `bootstrap_data_generator` that reads script files and dispatches them as source change events through the ETF's dispatchers. This means the ETF is responsible for converting script data into the right format, managing the bootstrap phase, and coordinating the handoff to streaming changes. The Drasi engine (drasi-lib or drasi-server) sees these as regular source change events — it has no awareness that a bootstrap is happening.

Drasi Server and drasi-lib now support a native `scriptfile` bootstrap provider — a Drasi plugin that loads initial data from JSONL files directly during query startup, before streaming begins:

```yaml
# Drasi server-config.yaml or embedded drasi config
sources:
  - id: facilities-db
    source_type: application
    auto_start: true
    bootstrapProvider:
      kind: scriptfile
      filePaths:
        - /data/initial_nodes.jsonl
        - /data/initial_relations.jsonl
```

The JSONL format uses typed records (`Header`, `Node`, `Relation`, `Finish`). Any source kind can use this bootstrap provider — it decouples bootstrap data loading from the source type.

For drasi-lib and drasi-server tests, we should migrate from the ETF's `Script` bootstrap mechanism to the native `scriptfile` bootstrap provider. This has several advantages:

- **Tests the real bootstrap path.** When users deploy Drasi with a `scriptfile` bootstrap, the query engine loads data through the same code path the test exercises. The current ETF approach tests a synthetic path that no real deployment uses.
- **Simpler ETF config.** The ETF no longer needs a `bootstrap_data_generator` for these tests — it only handles streaming changes after bootstrap. The Drasi engine owns the full bootstrap lifecycle.
- **Consistent with production.** The `scriptfile` provider handles format parsing, element construction, and bootstrap sequencing. Testing through it validates that pipeline end-to-end.

For drasi-platform (Kubernetes), the `scriptfile` provider is harder to use because the JSONL files need to be accessible from within the pod (e.g., via a ConfigMap, PersistentVolume, or init container). The ETF's current approach of streaming bootstrap data through dispatchers remains more practical for platform tests.

The migration involves:
1. Converting existing ETF bootstrap script files to the `scriptfile` JSONL format (`Header`, `Node`, `Relation`, `Finish` records)
2. Adding `bootstrapProvider: { kind: scriptfile, filePaths: [...] }` to the Drasi engine config files (`drasi-memory.yaml`, `server-config.yaml`, etc.)
3. Removing the `bootstrap_data_generator` block from the ETF's source config for these tests
4. Hosting the JSONL bootstrap files locally in the repo (for drasi-lib/server) or on Hugging Face Hub (for larger datasets)

### Using Drasi's Mock Source vs ETF Data Generators

Drasi Server and drasi-lib include a built-in Mock source (`kind: mock`) that generates synthetic data internally — `Counter`, `SensorReading`, and `Generic` types. For simple test scenarios that only need single-node data (projections, filters, property updates), the Mock source could replace the ETF's model data generator entirely. This would remove the ETF from the data path, making the test exercise Drasi's own data generation → query → reaction pipeline with nothing in between.

However, the Mock source currently only generates flat node data with no relationships or hierarchical structure. It cannot produce the `Building → Floor → Room` graph with `PART_OF` relationships that is needed for join queries, multi-hop traversals, or aggregations across a hierarchy. For these tests, the ETF's `BuildingHierarchy` model generator (or script-based sources) remains necessary.

If the team wants to reduce dependency on the ETF for data generation, one option would be to extend the Mock source to support relationship generation and hierarchical structures. This is a discussion point for the team — the trade-off is implementation effort in drasi-server/drasi-lib vs the testing simplicity gained by keeping data generation inside the Drasi engine.