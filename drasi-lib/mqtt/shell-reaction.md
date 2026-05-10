# Shell Reaction Design Document

* Project Drasi - 2026 Apr 24 - Ahmed Kamal (@ahmed-kamal2004)

## Overview

Shell Reaction is a Drasi Reaction component that executes operating system commands in response to continuous query result changes. This enables users to connect Drasi detections and state transitions to operational automations.

This design targets drasi-core reactions and aligns with the Drasi reaction model: Source -> Query -> Reaction.

## Terms and definitions

| Term | Definition |
|------|------------|
| Shell Reaction | A Drasi Reaction that executes configured system commands based on query change events |
| Reaction event | A single `ResultDiff` event emitted by a Query (`Add`, `Update`, `Delete`, `Aggregation` and `Noop`) |
| Query Result| A single `QueryResult` including the query id, timestamp, and vector of Reaction events |
| Execution policy | Configuration that constrains command path, arguments, environment vars, input data templates, timeout, and concurrency |

### User scenarios

- An operator runs continuous queries that detect abnormal conditions (for example sustained threshold violations). They configure Shell Reaction to invoke an operational script that creates a ticket, email or invokes internal/external endpoint.

- An application engineer wants to integrate Drasi with an existing internal tool that only exposes a CLI. Instead of building a new reaction service, they configure Shell Reaction to run the tool with event-derived arguments.

### Goals

1. Deliver a production-grade Secure Shell Reaction.
2. Support event-to-command mapping with explicit templates.
3. Provide runtime controls: timeout and max concurrency.
4. Ensure integration with current Drasi components.
5. Testing coverage for unit and integration tests.

## Design requirements

### Requirements

1. Must run as a Drasi Reaction component and consume query result change events from the continuous query.
2. Must validate configuration at startup and fail fast on invalid command templates or policy settings.
3. Must support both executable input modes:
	- stdin piped from parent process
	- environment variables
4. Must enforce the Single Binary Rule: `tokio::process::Command` executes one binary path from `command`.
5. Must validate executable on startup (exists, executable bit/permissions).
6. Must support non-blocking spawn execution.
7. Must support `max_concurrent` runners.
8. Must enable `kill_on_drop` option for spawned processes.
9. Must include unit tests and integration tests.

### Dependencies

- drasi-core reaction implementation patterns.
- Continuous query engine output mapping.
- Linux.

### Out of scope
- Automatic retries / circuit-breaker (scripts handle their own retry logic)
- Seccomp integration (operators should use OS-level constraints like cgroups, systemd MemoryMax, prlimit)
- Env clearance / minimal path (operators manage their env in the global env config or inside the script)
- Resource limits beyond timeout (same OS-level constraints)
- Privilege dropping / run-as user (operators should use sudo or setuid wrappers)
- Windows support (V1 targets Linux and macOS only; SIGTERM/SIGKILL and process groups assume POSIX)
- Secret injection via IdentityProvider (pass secrets through global env or fetch at runtime in the script)
- PATH-based executable resolution (require absolute paths in V1)
- Shell interpreter mode (operators who need piping can set executable to /bin/sh with static args ["-c", "<pipeline>"], which is safe because args are not templated)
- Broadcast drop detection (drops happen in the dispatch layer before reaching the reaction. Once drasi-lib stamps result diffs with sequence numbers, the reaction could detect gaps and expose a counter)
- Stdin opt-out (`enable_stdin: false`) for scripts that only need env vars and do not read stdin. V1 always sends stdin when a TemplateSpec exists (rendered template or raw JSON fallback), matching the HTTP reaction. Scripts that do not need stdin can simply not read it. If profiling shows the pipe write is a bottleneck on high-throughput edge deployments, a future version can add an opt-out flag

## Design

### High-level design

Shell Reaction receives query events and executes one process per event. Execution is performed with non-blocking spawn and sized concurrency.

Executable data ingress is supported in two modes:
1. `stdin` mode: event-derived payload is piped to child stdin.
2. `env` mode: selected event fields are projected to environment variables.

At startup, the reaction validates the configured executable path and enforces the Single Binary Rule.

### Architecture Diagram

```text
Query Container events
				|
				v
Shell Reaction
				v
Configured executable
	- stdin pipe from parent (optional)
	- env vars from event mapping (optional)
```

### Detail design

#### Configuration Validation

##### Command validation:

At startup, the reaction performs validation of the configured executable path:
1. Check that the path is absolute.
2. Check that the path exists.
3. Check that the path points to a file, not a directory.
4. Check that the file has executable permissions.

##### Configuration validation:

Although for each query config, we check the following:
1. check if the user configured a `route` for a query that the reaction isn't subscribed to. (Error)
2. check if the reaction is subscribed to a query that doesn't have a `route` configured and a `default_template` doesn't exist. (Error - the reaction MUST fail to start)

If there are configured env variables, they are validated against Linux variable naming rules, which are as follows (Error if any variable name doesn't follow these rules):
- Must start with a letter or underscore. 
- Can contain letters (small - a-z, capital - A-Z), digits, and underscores.
- Cannot contain spaces or special characters.

Final regex for validation of the env var name: `[A-Za-z_][A-Za-z0-9_]*`

And a warning is emitted if any of the configured env variables overlap with the parent process environment variables. (Warning)

#### Query Results handling and mapping

For empty `QueryResult` that contains control signals like `bootstrapStarted`, `bootstrapCompleted`, `running` and others, they are ignored with a debug log. (Debug)

For each `ResultDiff` event operation in the `QueryResult` based on the `QueryConfig` structure:

`QueryConfig` structure:
```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(bound(deserialize = "T: Deserialize<'de> + Default"))]
pub struct QueryConfig<T = ()>
where
    T: Default,
{
    /// Template specification for ADD operations (new rows in query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub added: Option<TemplateSpec<T>>,

    /// Template specification for UPDATE operations (modified rows in query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub updated: Option<TemplateSpec<T>>,

    /// Template specification for DELETE operations (removed rows from query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub deleted: Option<TemplateSpec<T>>,
}
```

`ResultDiff` structure:
```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type")]
pub enum ResultDiff {
    #[serde(rename = "ADD")]
    Add { data: serde_json::Value },
    #[serde(rename = "DELETE")]
    Delete { data: serde_json::Value },
    #[serde(rename = "UPDATE")]
    Update {
        data: serde_json::Value,
        before: serde_json::Value,
        after: serde_json::Value,
        #[serde(skip_serializing_if = "Option::is_none")]
        grouping_keys: Option<Vec<String>>,
    },
    #[serde(rename = "aggregation")]
    Aggregation {
        before: Option<serde_json::Value>,
        after: serde_json::Value,
    },
    #[serde(rename = "noop")]
    Noop,
}
```

- if `ADD` the `added` template is applied
- if `UPDATE` or `Aggregation` (same as Http reaction) the `updated` template is applied
- if `DELETE` the `deleted` template is applied
- if `Noop`, it is ignored with a debug log. (Debug)

#### Executable input modes

Shell Reaction supports both delivery channels for event data:
1. `stdin` mode: serialize selected payload template and pipe into child stdin.
2. `env` mode: map selected event fields to environment variables.

Notes:
- Both modes can be enabled together.
- If the env variable templated value is empty after rendering, the variable will be set with an empty string value in the child environment.

we will define a `ShellExtension` that will be holding the optional Env vars:
```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct ShellExtension {
    pub env: Option<HashMap<String, String>>,
}
```

which will be used in the `TemplateSpec` as an extension.

executables are defined in a new struct named `ShellCommand` which will be used in the `ShellReactionConfig` struct as part of the query-specific configuration.
```rust
pub struct ShellCommand {
  pub executable: String, // the command to execute, for example: "/bin/python3"
  pub args: Vec<String>, // the arguments to pass to the command, for example: ["main.py"]
}
```

Snippet of the `ShellReactionConfig` :
```rust
  /// Query-specific template configurations
  #[serde(default)]
  pub routes: HashMap<String, (ShellCommand, QueryConfig)>, 
  // where the key of the map is the query id, and the value is a tuple of (ShellCommand, QueryConfig)
```

Example config for both modes together:
```rust
let spec_add = TemplateSpec<ShellExtension> {
    template: "[{{query_name}}] + {{after.floor}}: {{after.temp}}".to_string(),
    extension: ShellExtension {
        env: Some(HashMap::from([
            ("FLOOR".to_string(), "{{after.floor}}".to_string()),
            ("TEMP".to_string(), "{{after.temp}}".to_string()),
        ]),
        )
    },
};

let spec_update = TemplateSpec<ShellExtension> {
    template: "[{{query_name}}] ~ {{after.floor}}: {{before.temp}} {{after.temp}}".to_string(),
    extension: ShellExtension {
        env: Some(HashMap::from([
            ("FLOOR".to_string(), "{{after.floor}}".to_string()),
            ("TEMP".to_string(), "{{after.temp}}".to_string()),
            ("BEFORE_TEMP".to_string(), "{{before.temp}}".to_string()),
        ]),
        )
    },
};

let spec_delete = TemplateSpec<ShellExtension> {
    template: "".to_string(), // empty string template - will send empty string to stdin
    extension: ShellExtension {
        env: Some(HashMap::from([
            ("FLOOR".to_string(), "{{before.floor}}".to_string()),
            ("TEMP".to_string(), "{{before.temp}}".to_string()),
        ])),
    },
};

let query_config = QueryConfig {
    added: Some(spec_add),
    updated: Some(spec_update),
    deleted: Some(spec_delete),
};

let executable = ShellCommand {
    executable: "/bin/python3".to_string(),
    args: vec!["/scripts/handle_event.py".to_string()],
};
let query_tuple = (executable, query_config);
// then inserted to the shell reaction config routes map with the query id as a key.
```

Missing `TemplateSpec` for an operation type (for example, missing `added` template) is allowed, and the reaction will fall back to a `default_template` if one is provided.

If no `default_template` is provided either, the reaction MUST fall back to sending the raw JSON representation of the event to stdin. No additional flag is needed for this behavior.

#### Default JSON Fallback

When a route or default_template exists for a delta type but omits the `template` field, the reaction sends the raw `QueryResult` JSON to stdin. This matches the HTTP reaction's behavior with missing body templates and provides a zero-config experience for scripts that just want to parse JSON.
If the template is provided but renders to an empty string, the reaction still sends the empty string to stdin (this is not the same as a missing template).

#### Global Env configuration

In addition to the env variable mapping per query, we can also support global env variable mapping for all spawned processes by adding a new field in the `ShellReactionConfig` struct named `env` which is a map of env variable name to template string.

```rust
pub struct ShellReactionConfig {
    pub max_concurrent: u32,
    pub timeout_s: u64,
    pub kill_on_drop: bool,
    pub max_stdin_bytes: usize, // max bytes to render and send to stdin
    pub capture_limit: usize, // max bytes to capture from stdout and stderr
    pub max_recent_invocations: u32, // for runtime inspection
    pub env: Option<HashMap<String, String>>, // global envs
    pub routes: HashMap<String, (ShellCommand, QueryConfig<ShellExtension>)>,
    pub default_template: Option<QueryConfig<ShellExtension>>,
}
```

Example config:
```yaml
env:
  GLOBAL_VAR1: "{{query_name}}"
  GLOBAL_VAR2: "static_value"
```
And it also supports the same validation rules as the query-specific env variable mapping.

#### Stdin data size limits

Configuration:
- `max_stdin_bytes` specifies the maximum number of bytes to render and send to stdin (default 1 MiB).

The reaction MUST enforce a configurable maximum stdin payload size (`max_stdin_bytes`, default 1 MiB). If the rendered template output exceeds this limit, the reaction MUST reject the event without spawning the child process, log a warning, and increment a drop counter. (Warning)

#### Trailing newlines in data

The reaction checks if trailing newlines `\n` in the rendered template output for stdin data. If trailing newlines aren't detected, they are automatically appended to ensure proper formatting for commands that expect newline-terminated input. (Debug log)

#### Stdout and stderr capture and limits

To prevent excessive memory usage, the reaction captures only the first `capture_limit` bytes of stdout and stderr from the child process. If the output exceeds this limit, it is truncated and a warning is logged. (Warning)

Regarding the stdout, it is logged at the `debug` level, while stderr is logged at the `warn` level.

#### Runtime inspection and debugging

Configuration:
- `max_recent_invocations` specifies how many recent execution results to keep in memory for inspection.

The reaction MUST maintain a ring buffer of recent execution results in memory, exposed through the `properties` field in the `ReactionRuntime` struct (via the `get_reaction_info` / `properties()` method on the `Reaction` trait).

This ring buffer is implemented as a `VecDeque` with a configurable capacity (`max_recent_invocations`, default 10). New execution results are pushed to the back and old results are popped from the front when the capacity is exceeded.

This is always enabled (not gated behind a configuration flag). On edge devices without centralized log aggregation, this is often the only way to debug script failures. The memory cost is negligible.

Each stored execution result includes:
- timestamp start of execution
- timestamp end of execution
- exit code
- stdout (already truncated by the `capture_limit` configuration)
- stderr (already truncated by the `capture_limit` configuration)

Note: the `properties` field in the `ReactionRuntime` is a mirror for the `properties` method in the `Reaction` trait.

Current schema of the `ReactionRuntime` struct,
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReactionRuntime {
    /// Unique identifier for the reaction
    pub id: String,
    /// Type of reaction (e.g., "log", "http", "grpc", "sse", "platform")
    pub reaction_type: String,
    /// Current status of the reaction
    pub status: ComponentStatus,
    /// Error message if status is Error
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error_message: Option<String>,
    /// IDs of queries this reaction subscribes to
    pub queries: Vec<String>,
    /// Reaction-specific configuration properties
    pub properties: HashMap<String, serde_json::Value>,
}
```
#### Runtime model

`spawn` as non-blocking and tracks in-flight children.

Example flow:
```rust
let mut child = Command::new(&executable.executable)
    .args(&executable.args)
    .envs(env_mapping) // set environment variables from mapping
    .stdin(Stdio::piped())
    .spawn()?;

let mut stdin = child.stdin.take().expect("Failed to open stdin");

tokio::spawn(async move {
    // write event data to stdin
    stdin.write(event_mapped_payload.as_bytes()).await.expect("Failed to write to stdin");
    drop(stdin);
});

// wait for exit
let _ = child.wait().await.expect("Failed to wait for child process");

// get exit status and output.


```

Controls:
1. `max_concurrent` limits active child processes.
2. queue/backpressure policy is applied when at capacity.
3. `kill_on_drop` can be enabled so process is terminated when handle is dropped during shutdown/restart.

#### Concurrency and backpressure

Configuration:
- `max_concurrent` limits the number of active child processes.

The reaction delegates event ingestion to `ReactionBase::enqueue_query_result()`, which enqueues events into the existing bounded priority queue using `enqueue_wait()`. Backpressure is handled upstream by the dispatch layer, not by the reaction itself:

- In **channel mode**, blocking propagates backpressure through the dedicated channel to the source.
- In **broadcast mode**, the broadcast channel drops lagged messages before they reach the reaction.

No intermediate unbounded channel is needed. The `max_concurrent` limit is enforced when dequeuing from the priority queue and spawning child processes. When the limit is reached, the reaction stops dequeuing until a running process completes.

#### Timeout and process termination

Configuration:
- `timeout_s` configuration field specifies the maximum allowed execution time for the child process in seconds. If the process exceeds this time, it is terminated.


Termination is performed in two stages:
1. Send `SIGTERM` to the whole process group to allow graceful shutdown.
2. If the process doesn't exit within a grace period (for example, `timeout_s`/ 8), send `SIGKILL` to force termination.

Writing to stdin and waiting for process exit are performed concurrently under the same timeout to prevent deadlocks where the process is waiting for stdin data while the reaction is waiting for process exit.

Initial semi-pseudo code for timeout implementation (stdout and stderr capture and limits are not included in this snippet, in implementation, they will be treated as another more task that runs concurrently with the write and wait tasks, and also under the same timeout):

```rust
use tokio::time::{timeout, Duration};
use tokio::io::AsyncWriteExt;
use std::os::unix::process::CommandExt;

// Spawn child in its own process group so kill(-pgid, ...) targets
// the child subtree, not Drasi's own process group.
let mut child = Command::new(&executable)
    .args(&args)
    .envs(&env_mapping)
    .stdin(Stdio::piped())
    .process_group(0)  // new process group with child PID as PGID
    .kill_on_drop(true)
    .spawn()?;

let pgid = child.id().unwrap() as i32;
let timeout_duration = Duration::from_secs(timeout_s);

let write_fut = async {
    if let Some(mut stdin) = child.stdin.take() {
        stdin.write_all(&payload).await?;
        stdin.shutdown().await?;
    }
    Ok::<_, anyhow::Error>(())
};

let wait_fut = async {
    let status = child.wait().await?;
    Ok::<_, anyhow::Error>(status)
};

// Both run concurrently under one timeout
let result = timeout(timeout_duration, async {
    let (write_result, wait_result) = tokio::join!(write_fut, wait_fut);
    write_result?;
    wait_result
}).await;


match result {
    Ok(Ok(status)) => {
        Ok(status)
    }
    Ok(Err(e)) => {
        Err(e)
    }
    Err(_) => {

        //..........send SIGTERM to the whole process group
        unsafe {
            libc::kill(-pgid, libc::SIGTERM);
        }

        //..........wait for a grace period for the process to exit gracefully
        match timeout(Duration::from_millis(timeout_s * 1000 / 8), child.wait()).await {
            Ok(Ok(status)) => Ok(status),

            _ => {
                //..........if the process didn't exit, send SIGKILL to the whole process group
                unsafe {
                    libc::kill(-pgid, libc::SIGKILL);
                }

                let _ = child.wait().await;
                Err(anyhow::anyhow!("process killed after timeout"))
            }
        }
    }
}
```

#### Failure retries [Out of scope for V1, but can be added in future iterations]

1. if child process exits with non-zero code, classify as failure and apply retry policy if configured.

`retry_on_failure` example config:
```yaml
executable: "/bin/python3"
args: ["/scripts/handle_event.py"]
retry_on_failure:
  enabled: true
  max_retries: 3
  base_delay_ms: 1000 ## ( ms base ) exponential backoff with 2 multiplier
```

#### Seccomp implementation proposal [Out of scope for V1, but can be added in future iterations]

This section follows the intended Rust model using `tokio::process::Command`, `std::os::unix::process::CommandExt::pre_exec`, and `seccompiler::apply_filter`.

Design behavior:
1. Build a seccomp filter before spawn using a selected set of capabilities or profile.
2. Apply the filter in `pre_exec` (child process after fork, before exec).

##### Models
- Capabilities model:
Giving capabilities based on configuration, for example: `read_files`, `write_files`, `network`, `delete_files` and so on.

- Profile model:
common used capabilities can be grouped into profiles together. For example, `logger` profile can include `write_files` capability, and `changer` profile can include `write_files` and `delete_files` capabilities.

Config shape (proposal):

```yaml
executable: "/bin/python3"
args: ["/scripts/handle_event.py"]
hardening:
  seccomp:
    enabled: true
    capabilities:
      - read_files
      - write_files
  process:
    env_clear: true
    minimal_path: "/usr/local/bin:/usr/bin"
```

Implementation notes:
1. This feature is only present in Linux.

#### Env variable clearance [Out of scope for V1, but can be added in future iterations]

To minimize inherited environment variable risks, we propose an `env_clear` option that starts with an empty environment for the child process. The reaction configuration can then specify a set of needed variables from parent environment within the execution context.

example config:
```yaml
executable: "/bin/python3"
args: ["/scripts/handle_event.py"]
hardening:
  process:
    env_clear: true
    envs: [PATH, CUSTOM_VAR]
```


### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable.

- Describe the REST APIs in detail for new resource types or updates to existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components. -->

ShellReactionConfig example:

```yaml
max_concurrent: 5
capture_limit: 1024
max_stdin_bytes: 1048576
timeout_s: 10
kill_on_drop: true
env:
  LOG_LEVEL: "DEBUG"
  DEPLOYMENT: "edge-gateway-01"
routes:
  overheated-machines:
    command:
      executable: "/usr/bin/python3"
      args: ["/opt/scripts/overheat_handler.py"]
    config:
      added:
        template: '{"alert": "new", "machine": "{{after.id}}"}'
        env:
          ALERT_TYPE: "NEW_OVERHEAT"
      updated:
        template: '{"alert": "update", "temp": {{after.current_temp}}}'
        env:
          ALERT_TYPE: "TEMP_CHANGE"
default_template:
  added:
    template: '{"event": "added", "data": {{json after}}}'
  updated:
    template: '{"event": "updated", "data": {{json after}}}'
  deleted:
    template: '{"event": "deleted", "data": {{json before}}}'
```

## Security

Security model in this design prioritizes constrained execution:

1. Single executable path with startup validation reduces command injection surface.
2. stdin/env mappings are explicit and template-driven.
3. `kill_on_drop` helps optional avoidance of orphaned processes on crash/redeploy paths.

Out of scope hardening features that can be added in future iterations:
1. seccomp profile and Linux capability reduction
2. `env_clear` to start with a minimal environment.

## Supportability

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->
Metrics will include:
- timeout firings
- non-zero exits
- stdout truncations
- stderr truncations
- stdin payload size rejections
- active processes (current concurrency gauge)
- results processed (can be grouped by operation type)

And they are exposed through the `properties` method in the `Reaction` trait implementation for the Shell Reaction, and also each event is logged using the `log` crate with appropriate log levels.

Levels for logs:
- timeout firings: warn
- non-zero exits: warn
- stdout truncations: warn
- stderr truncations: warn
- stdin payload size rejections: warn
- active processes (current concurrency gauge): debug
- results processed (can be grouped by operation type): debug

All non-normal events (timeouts, non-zero exits, truncations, rejections, drops) are logged at the `warn` level to ensure visibility in production environments. Normal operational metrics (active processes, results processed) are logged at the `debug` level to avoid noise.

## References

<!-- Optional. Add the design documents and references you use for this document. -->
- https://www.stackhawk.com/blog/rust-command-injection-examples-and-prevention/
- https://ssojet.com/escaping/shell-escaping-in-rust#safe-command-execution-in-rust
- https://mojoauth.com/escaping/shell-escaping-in-rust#real-world-examples-for-shell-escaping-using-rust
- https://docs.rs/seccompiler/latest/seccompiler/