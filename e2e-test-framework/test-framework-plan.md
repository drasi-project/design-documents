# Test Framework Plan

* Project Drasi — April 29, 2026 — Ruokun Niu (@ruokun-niu)

## Test Framework Requirements


**Runnable manually.** Developers must be able to run the full test suite from their local machine or as a Github Actions workflow (triggered via `workflow_dispatch`). The existing building_comfort examples in the [test-infra](https://github.com/drasi-project/test-infra) repo demonstrate three distinct run patterns:

- **drasi-lib (in-process):** A single shell script runs the test-service with drasi-lib compiled in — no external processes needed. The script invokes `cargo run --release --manifest-path ./test-service/Cargo.toml -- --config <config.json>` from the `e2e-test-framework` directory. Everything runs in one process.
- **Drasi Server (standalone):** A shell script first builds and starts the drasi-server binary (from a sibling `../../drasi-server` directory) with a `server-config.yaml`, waits for its health check, and then starts the test-service with a separate `config.json` that dispatches changes via HTTP or gRPC to the running server. Currently it can use either a prebuilt binary or execute `cargo run` from a `drasi-server` repo. It is also worth noting that we have four variants of the drasi-server pattern: http, http adaptive, grpc and grpc adaptive.
- **Drasi Platform (Kubernetes):** Assumes an already-running Kubernetes cluster (the user supplies the kubeconfig; the example does not provision one). The test-service is deployed as a pod alongside Drasi Platform, and the run script uses `kubectl port-forward` to expose the test-service's API endpoint locally. Tests are then controlled via REST API calls (using `curl` or `.http` files) against that port-forwarded endpoint. Each query container index backend is a separate example directory with its own self-contained config (`query_container_default`, `query_container_memory`, `query_container_redis`, `query_container_rocks`).

Each example provides both `run_debug.sh` and `run_release.sh` scripts. For CI, the same scripts (or equivalent `workflow_dispatch` triggers) can be used in GitHub Actions.

**Automatable in CI workflows.** The test framework should be executed in automated tests. There are multiple ways to approach this:

- **Triggered via `workflow_dispatch`.** A manually triggered workflow that lets the user select which target mode and variant to run. This is useful for on-demand testing — for example, running the full suite against a specific drasi-server release before cutting a new version, or validating a particular index backend after a related code change.

- **Triggered on pull requests or pushes to `main`.** Running tests automatically on PRs. The trade-off is increased build time. Compiling the test-service (especially for drasi-lib mode, which compiles all of drasi-lib into the binary) and running 100K+ event tests adds significant minutes to each PR check. A practical compromise is to run a smaller, faster subset (e.g., 1K events with memory index) on every PR and reserve the full suite for `workflow_dispatch` or merges to `main`.

- **Scheduled runs (cron).** A nightly or weekly schedule can run the full test suite against the latest `main` of drasi-core, drasi-server and drasi-platform.


#### Current Config Structure

Today, each test target has its own completely separate config file with the test logic (data model, queries, etc.) duplicated inside it. 

The core test logic — the BuildingHierarchy model config (seed, change_count, sensor definitions), the query text, and the stop trigger count — is duplicated across every target variant. Adding a new test scenario or modifying an existing one requires updating 3-4 files. The `source_change_dispatchers`, `output_handler`, and `drasi_servers` blocks are the only parts that differ between targets.

#### Proposed Changes

The goal is to have a single central test config per test scenario that works with any Drasi target. The test config defines the "what" (test ID, data model, queries, stop criteria), and a separate pattern file defines the "how" (dispatchers, handlers, engine/server config, runtime settings). The ETF merges them at load time.

Each test scenario has one central config and multiple pattern files:

```
building_comfort/
├── test-config.yaml                   # Central: test ID, data model, stop triggers
├── patterns/
│   ├── drasi-lib-memory.yaml          # drasi-lib with memory index
│   ├── drasi-lib-rocksdb.yaml         # drasi-lib with RocksDB index
│   ├── server-grpc-memory.yaml        # drasi-server via gRPC, memory index
│   ├── server-grpc-rocksdb.yaml       # drasi-server via gRPC, persistent index
│   ├── server-http-memory.yaml        # drasi-server via HTTP, memory index
│   ├── platform-memory.yaml           # drasi-platform, memory query container
│   ├── platform-redis.yaml            # drasi-platform, Redis query container
│   └── platform-rocksdb.yaml          # drasi-platform, RocksDB query container
```

To run a test, you pass both files:

```bash
cargo run -p test-service -- --test-config test-config.yaml --pattern patterns/drasi-lib-memory.yaml
```

The **test config** (`test-config.yaml`) contains only the target-agnostic parts — the data model generator, query definitions, reaction observers, and stop triggers. Dispatchers and handlers are left empty for the pattern file to fill in:

```yaml
# test-config.yaml — central test definition, same for all targets
test_id: building_comfort
version: 1
description: Building comfort monitoring with room temperature, humidity, and CO2

sources:
  - test_source_id: facilities-db
    kind: Model
    source_change_dispatchers: []       # filled in by pattern file
    model_data_generator:
      kind: BuildingHierarchy
      seed: 123456789
      change_count: 100000
      spacing_mode: none
      time_mode: "2025-01-03T10:03:15.4Z"
      building_count: [1, 0]
      floor_count: [1, 0]
      room_count: [1, 0]
      room_sensors:
        - kind: NormalFloat
          id: temperature
          momentum_init: [5, 1, 0.5]
          value_change: [1, 0]
          value_init: [5000, 0]
          value_range: [0, 10000]
        # ... co2, humidity sensors ...

reactions:
  - test_reaction_id: building-comfort
    output_handler: {}                  # filled in by pattern file
    stop_triggers:
      - kind: RecordCount
        record_count: 90000
```

A **pattern file** for drasi-lib (`patterns/drasi-lib-memory.yaml`) supplies the dispatchers, handlers, engine config, and runtime settings:

```yaml
# patterns/drasi-lib-memory.yaml
drasi_servers:
  - id: drasi-lib-instance
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

source_dispatchers:
  facilities-db:
    kind: DrasiServerChannel
    drasi_server_id: drasi-lib-instance
    source_id: facilities-db
    buffer_size: 2048

reaction_handlers:
  building-comfort:
    kind: DrasiServerChannel
    drasi_server_id: drasi-lib-instance
    reaction_id: building-comfort-alerts
    buffer_size: 1024

run:
  drasi_servers:
    - test_drasi_server_id: drasi-lib-instance
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

A **pattern file** for drasi-server (`patterns/server-grpc-memory.yaml`) uses gRPC dispatchers with no embedded engine:

```yaml
# patterns/server-grpc-memory.yaml
source_dispatchers:
  facilities-db:
    kind: Grpc
    host: localhost
    port: 50051
    source_id: facilities-db
    timeout_seconds: 60

reaction_handlers:
  building-comfort:
    kind: Grpc
    host: 0.0.0.0
    port: 50052
    query_ids: [all-rooms]
    correlation_metadata_key: x-query-sequence

# The referenced server-config-memory.yaml must declare a query with id `all-rooms`
# (matching the query id used by all patterns for this test scenario).
drasi_server_config: server-config-memory.yaml

run:
  sources:
    - test_source_id: facilities-db
      start_mode: auto
  reactions:
    - test_reaction_id: building-comfort
      start_immediately: true
      output_loggers:
        - kind: JsonlFile
        - kind: PerformanceMetrics
```

**How the merge works.** When the ETF loads both files, it:

1. Takes the test config as the base (sources, reactions, stop triggers, data model)
2. Injects `source_dispatchers` from the pattern file into each source by matching on `test_source_id`
3. Injects `reaction_handlers` from the pattern file into each reaction by matching on `test_reaction_id`
4. Injects the `drasi_servers` block (if present) into the test definition
5. Uses the `run` section from the pattern file as the `test_run_host` config
6. Wraps both in the `data_store` / `test_repos` boilerplate automatically

**What this enables:**

- Adding a new test scenario means writing one `test-config.yaml` — no per-target duplication
- Adding a new target pattern means writing one pattern file — no test logic duplication
- Changing a test's data model or query updates one file that all patterns share
- The pattern files are small and focused (dispatchers + engine config only)

**ETF changes required:**

- Replace the existing `--config` CLI argument with `--test-config <path>` and `--pattern <path>`. The existing self-contained config files will be migrated to the new two-file layout as part of this change.
- A config merge function that combines the two files as described above

---

## How Each Target Is Consumed

This section describes the **proposed** approach for how the ETF should consume each target going forward. The "Test Framework Requirements" section above describes today's behavior; this section describes where we want to land.

**drasi-lib.** The ETF consumes drasi-lib via the `drasi-core` git submodule in the `test-infra` repo. The `test-run-host` crate depends on it as a Cargo path dependency, so drasi-lib is compiled directly into the test-service binary. Tests use `DrasiServerChannel` dispatchers for zero-network in-process communication. To test a different version, update the submodule pointer.

**drasi-server.** The drasi-server binary is obtained in one of two ways depending on the trigger:

- **Release validation and scheduled runs.** The run script pulls a pre-built drasi-server binary from GitHub Releases (or the published Docker image). This exercises the same artifact users consume.
- **`pull_request` runs.** The run script builds drasi-server from source with `cargo run` against a checked-out `drasi-server` repo (typically a sibling working copy or a checkout pinned by the workflow). This lets PRs against drasi-server validate changes before any release exists.

In both cases the server is started with a `server-config.yaml` (referenced by the pattern file's `drasi_server_config` field), the script waits for its health check, and then the ETF test-service runs separately and dispatches changes to the server via HTTP or gRPC as specified in the pattern file.

**drasi-platform.** The deployment model stays the same as today (test-service deployed as a pod alongside Drasi Platform, controlled via `kubectl port-forward` to its REST API). The proposed changes are: (1) the framework optionally provisions a local kind cluster automatically when no kubeconfig is provided, so runs do not require an already-running cluster — for CI this becomes the default since it needs no external infrastructure; (2) the per-index-backend example directories (`query_container_memory`, `query_container_redis`, `query_container_rocks`, ...) are collapsed into a single test config plus per-backend pattern files (`platform-memory.yaml`, `platform-redis.yaml`, `platform-rocksdb.yaml`). Provisioning and tearing down a managed cluster (e.g., AKS) within a GitHub Actions workflow is possible but adds a lot of complexity, so it is not part of the default flow.

---

## Hosting Test Configs and Test Data

All test configuration and test data must be publicly accessible without authentication tokens. The ETF's `GitHub` test repo backend fetches files via the GitHub REST API at runtime, which is subject to rate limiting (60 requests/hour unauthenticated, 5,000 with a PAT). This makes it unsuitable for hosting test data that gets pulled repeatedly across test runs and CI jobs. The options are:

- **Local storage in the repo (recommended for configs and small data):** Test configs, Drasi engine configs, and small golden files live directly in the `test-infra` repo. After cloning (or checking out in CI), the ETF uses the `LocalStorage` backend with a relative filesystem path — no API calls, no rate limits. This works well for YAML configs and small expected-result files. Smaller synthetic datasets (generated by model sources with fixed seeds) also fit here since they are produced at runtime and don't need to be stored.

- **Hugging Face Hub (recommended for test data files):** Public datasets on Hugging Face are freely downloadable without authentication and without the rate limiting issues of GitHub's API. Files are accessible via a predictable HTTP URL pattern: `https://huggingface.co/datasets/{org}/{repo}/resolve/main/{path}`. We would create a public dataset repo (e.g., `drasi-project/test-data`) and push JSONL files to it. HF Hub is git-based and automatically uses Git LFS for large files, so versioning is built in. The ETF would download files with a simple HTTP GET to the resolve URL — no special client library needed. This is free, public, requires no infrastructure to maintain, and has no rate limits for normal usage.

The recommended approach is: **test configs and smaller-sized files in the repo** (LocalStorage), **JSONL test data files on Hugging Face Hub** (downloaded via HTTP at test time or pre-cached locally).

---

## Expanding the Test Suite

We need a defined test suite for drasi-platform, drasi-server, and drasi-lib that covers four dimensions: component configurations, query complexities, index configurations, and failure recovery.

### Component Configurations & Query Complexities

The test suite should exercise a variety of source, query, and reaction configurations. The scenarios to cover include:

- Different source and bootstrap types (model-generated synthetic data, script-replayed recorded data)
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

**NOTE:** This might be something to focus on after we have completed all of the resilience-related work

The following scenarios are feasible with the current ETF capabilities or with modest extensions:

- **Source pause and resume.** The ETF already supports `pause`/`start` control on sources via its REST API. A test can dispatch N events, pause the source, verify partial results, resume, and verify the final result matches the full run. This validates that no events are lost during a pause/resume cycle.

- **Query stop and restart.** The ETF supports `stop`/`start` on query observers. For drasi-lib mode, the Drasi engine can also stop and restart a query. A test can process events, stop the query, continue dispatching changes (which the query misses), restart the query (triggering re-bootstrap), and verify the query reaches the correct final state.

- **Server process restart (drasi-server).** In standalone mode, the run script can kill and restart the drasi-server binary between test phases. This requires the run script to orchestrate pause → kill → restart → resume, which is scriptable but not yet automated in the ETF.
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


#### Proposed reporting approach

After each test run, the ETF (or a post-processing script) should produce a single `test-report.json` that consolidates the key metrics from all loggers into one file. This makes it easy to compare runs, track regressions, and display in CI summaries.

   **Local runs.** The ETF already writes logger output to a `test_data_cache/` directory (configured via `data_store_path`). The `test-report.json` should be written alongside the other logger output in the same run directory — for example, `test_data_cache/<repo>/<test_id>/<run_id>/test-report.json`. This keeps all artifacts for a run co-located and easy to find. Developers can also specify a custom output path via a `--report-output <path>` CLI flag if they want reports written elsewhere.

   **GitHub Actions.** The `test-report.json` is written to the same run output directory within the runner's workspace. A post-run step parses the report and writes a Markdown results table (status, throughput, latency, correctness) into `$GITHUB_STEP_SUMMARY`, which renders directly on the workflow run's Summary tab — visible immediately without downloading anything. The same step uploads the entire run output directory (including `test-report.json`, JSONL logs, and profiler output) as a [build artifact](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/storing-and-sharing-data-from-a-workflow#uploading-build-and-test-artifacts) using `actions/upload-artifact` for historical tracking and debugging. Build artifacts are retained for 90 days by default and this is configurable.

---

## Additional Notes

### Replace ETF Script Bootstrap with the Drasi `scriptfile` Bootstrap Provider

Today, when a test uses recorded data (as opposed to model-generated data), the ETF handles bootstrapping itself: the `Script` kind source has a `bootstrap_data_generator` that reads script files and dispatches them as source change events through the ETF's dispatchers. This means the ETF is responsible for converting script data into the right format, managing the bootstrap phase, and coordinating the handoff to streaming changes. The Drasi engine (drasi-lib or drasi-server) sees these as regular source change events — it has no awareness that a bootstrap is happening.

Drasi Server and drasi-lib now support a native `scriptfile` bootstrap provider — a Drasi plugin that loads initial data from JSONL files directly during query startup:

```yaml
# Drasi server-config.yaml or drasi-lib config
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


### Workflow improvement
- Add the Scorecard and DevSkim workflows to the test-infra repository.