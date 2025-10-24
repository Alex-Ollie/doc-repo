# ben_sdk

The Ben SDK serves as the primary interface through which developers integrate their systems with Ben.
Its design centers on the question: “How can this be made truly adoptable?”

To achieve that, Ben’s engineering philosophy follows an outward-in approach: beginning with the outermost layer (the parts that interact directly with external systems) and progressively moving inward toward the core.
This methodology ensures that integration patterns, data formats, and developer ergonomics drive the system’s foundation rather than being retrofitted later.

By starting from the edge, the SDK defines the nature and structure of incoming data early, enabling the inner layers of Ben to evolve around real telemetry and use-cases rather than speculative assumptions.
An inward-out approach, by contrast, often results in brittle extensions and bolted-on features.

The following sections outline the Ben SDK’s architecture and the design rationale behind each component.

### Socket Centered development
To integrate seamlessly with external systems, Ben leverages Rust’s mature Foreign Function Interface (FFI) capabilities.
Through these interfaces, the SDK exposes a controlled mechanism for applications to open sockets and communicate with Ben sidecars using standard file descriptors.

Once data reaches a sidecar, it remains within Ben’s trusted execution boundary. From that point forward, communication across the ecosystem is handled through gRPC, ensuring authenticated, structured, and efficient message exchange between components.

This design cleanly separates external integration (via FFIs and sockets) from internal coordination (via gRPC), maintaining both flexibility for adopters and consistency across the Ben ecosystem. 


### Sockets and levels
| Socket        | Level                 | Action                                                                                   |
|----------------|-----------------------|------------------------------------------------------------------------------------------|
| diagnostic fd  | 1 – Basic telemetry   | Utilized to determine operational bounds                                                 |
| event fd       | 2 – Event telemetry   | Utilized to determine anomalies, determine if shifts need to be taken                    |
| cntrl fd       | 3 – Control channel   | Channel is akin to the "line to the president" where Ben issues commands                 |

Sockets originating from external applications are routed through a Ben sidecar, which forwards telemetry to the spine. The spine acts as the central message broker, distributing data to the appropriate detectors for analysis.

To maintain performance and separation of concerns, socket traffic is divided by purpose:

Diagnostic / Informational data — high-volume, low-priority signals delegated to a dedicated diagnostic detector.

Event and Control data — lower-volume, higher-impact signals routed to specialized detectors for real-time evaluation and command handling.

This division balances load across the detection layer and ensures that critical events are processed without being throttled by background telemetry.

Once processed, detector output — enriched by machine learning models — follows a hot path into the ClickHouse data silo, where it is stored for analytics, correlation, and long-term retention.

---

## Rust Implementation

The core design ethos of the Ben SDK is extensibility without cost — enabling maximum flexibility for developers while maintaining the structural constraints necessary for Ben to remain reliable and useful.

This balance is achieved through a declarative procedural macro approach.
The initial implementation began as an attribute macro, but this proved too restrictive for Ben’s needs. To support richer expression and composability, the design evolved into a domain-specific language (DSL) implemented as a declarative macro that leverages proc-macro under the hood.

This hybrid design provides the ergonomic simplicity of declarative macros while retaining the full expressive power of procedural generation, allowing schemas, telemetry definitions, and detectors to be defined cleanly and expanded safely at compile time.

#### How that looks

```rust
use ben_macros::ben;

ben! {
    // Required bindings — Ben wires these into generated code.
    client = client, hb = sampler, tx = telem, wake = notify, ctrl = control;

    //Heartbeat Cadences
    beats {
        // High-priority system pulse; strict liveness requirement.
        high every 10s
             purpose "liveness"
             coalesce latest
             priority 100;

        // Regular operational metrics; tolerant to deferral.
        normal every 30s
               purpose "ops"
               coalesce latest
               priority 50;

        // Diagnostic payloads: large, lossy, explicitly droppable.
        diag every 5m
             purpose "diagnostic"
             drop_first
             priority 10;
    }

    // Backpressure & Flush Policy 
    backpressure {
        flush_bytes 64KiB per_cycle;
        on_would_block { shed diag keep high; }
    }

    //Health Rules (Evaluated per Beat) 
    rules {
        // Derived state (degraded) is evaluated automatically.
        if degraded for 3 cycles then die enforce;
        if queue_depth > 80% for 2 cycles then backoff(rate = 0.5) warn;
        if retry_count >= 5 then raise_event "network_retry_excess" log_only;
    }

    //Control Channel Handlers
    control {
        verify ed25519;

        on "Quarantine" {
            emit diag purpose "pre_quarantine_report";
            disable beats normal, diag;
            enter quarantine;
        }

        on "Resume" {
            enable beats normal, diag;
            emit high purpose "post_quarantine_resume";
        }

        on "AdjustCadence" {
            update beats.normal every 10s;
        }
    }

    //Adaptive Behavior (Optional)
    adaptive {
        // When CPU > 90% for 3 cycles, slow Normal beats to 60s.
        if cpu_pct > 90 for 3 cycles { update beats.normal every 60s; }

        // When CPU < 40% for 5 cycles, restore Normal beats to 30s.
        if cpu_pct < 40 for 5 cycles { update beats.normal every 30s; }
    }
}
```

Lets break the top part down:
| Binding           | Expected Type / Trait                                                           | Purpose                                                                                                                                                                                          |
| ----------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `client = client` | something that implements `BenClient`                                           | The *brains* of the SDK. Handles queuing, coalescing, signing, and flushing telemetry through the telemetry channel. It’s the sidecar’s logic core.                                              |
| `hb = sampler`    | something that implements `ProvidesHeartbeat` *(optional trait layered on top)* | The source of heartbeat data — the struct in your subsystem that knows how to collect CPU%, RSS, timestamps, etc. The macro calls `sampler.sample(BeatClass::High)` to build heartbeat payloads. |
| `tx = telem`      | `TelemetryCh` struct (wraps a boxed `dyn Channel`)                              | The actual byte pipe for sending telemetry. Usually a socket or UDS connection to the Spine.                                                                                                     |
| `wake = notify`   | `NotifyCh` struct (wraps a `dyn Channel`)                                       | The local event line — a cheap “nudge” channel that wakes the I/O loop after new telemetry is ready to flush.                                                                                    |
| `ctrl = control`  | `ControlCh` struct (wraps a `dyn Channel`)                                      | The control lane — Ben’s “red phone.” Used to receive signed commands and send back acknowledgements.                                                                                            |

Three of the SDK’s core components implement the Channel trait object, which defines the fundamental I/O contract for Ben’s communication layer through its send() and recv() methods.
The upper layer of the SDK orchestrates channel creation and coordination, establishing the flow between telemetry producers, sidecars, and detectors.

The file descriptor (FD) model introduces important ownership semantics, implemented through three operational “lanes”:

Exclusive Ownership — The safest and simplest approach, where each channel has sole ownership of its FD. This eliminates contention and ensures deterministic cleanup, though it increases port consumption.

Shared Descriptors — Used sparingly, this allows controlled concurrency via Arc<Mutex<T>> when multiple readers or writers must access the same FD.

Duplicate Descriptors — A specialized case relying on OS-level dup() operations. It should be reserved for narrow scenarios, as interleaving reads and writes can degrade stability in large-scale systems.

At its core, the SDK’s communication model is built upon a minimal, composable set of abstractions — three traits, three structs, and a few key enumerations (excluding socket builders).
This structure keeps the implementation surface small while allowing the SDK to scale across environments and languages without loss of clarity or safety.

---

### Socket building for foreign interfaces

A central design question for the SDK is: “How do we build and expose the sockets necessary for communication?”

Since Ben’s architecture relies heavily on socket-based communication, socket construction must align with the conventions of each supported language runtime.
The current MVP targets Go, Python, and Node.js, ensuring smooth integration with systems that already operate in these ecosystems.

At a high level, the flow proceeds as follows:

```
ben_socket_init(runtime, handle, channel)
│
├─ if unix → build_unix()
│     ├─ match runtime → build_from_python / go / node
│     └─ wrap_channel(fd)
│
└─ if windows → build_windows()
      ├─ match runtime → build_from_python_win / go_win / node_win
      └─ wrap_channel(sock)
```
Each runtime uses its native facilities to obtain a socket handle (fd on Unix, SOCKET on Windows).
The SDK’s unified FFI layer then wraps that handle into a Rust-managed Channel, normalizing differences in handle types and ensuring ownership semantics remain consistent across operating systems.

This design keeps language integrations lightweight and uniform, while preserving Rust’s guarantees of safety and deterministic cleanup.

Since sockets are tied to structs which dictate what they are (diagnostic or telemetry, event and control), we are only worried about producing the socket at this part.
The structs that house them would look like:

```rust
pub struct TelemetryCh {
    #[cfg(unix)]
    fd: std::os::unix::io::RawFd,

    #[cfg(windows)]
    sock: std::os::windows::io::RawSocket,
}

```

These implement channel which then can be used for ben on whole.

---

## Schema

Schema design within the Ben SDK must balance extensibility and interoperability.
If schemas are too rigid, integrators are constrained to only what Ben natively supports, limiting adoption.
If they are too loose, Ben risks degenerating into an unstructured JSON junk drawer where telemetry loses meaning and comparability.

This balance is achieved through the use of derive macros, which enable declarative schema definitions while enforcing strict type and structure rules at compile time. Macros provide the flexibility developers need, without compromising schema integrity or analytic performance.

### Separation of Responsibilities

Schema generation is handled by a dedicated executable, independent of the SDK runtime.
Embedding schema-building logic directly within SDK development would be both unsafe and operationally rigid.
Instead, the schema compiler produces validated DDL (Data Definition Language) definitions and associated metadata that can be deployed through Ben’s Schema Registrar service.

### Safety in Schema Evolution

Thresholds and field metadata are allowed to change independently of the core SDK.
To prevent accidental or careless schema modifications, the schema builder enforces a multi-stage confirmation process — a deliberate “three layers of are-you-sure” approach. Each stage presents the pending changes and their impact, ensuring schema evolution remains intentional and auditable.

Handling schema modifications for high-volume tables remains a complex challenge. Altering active tables at scale can cause significant ingestion latency or locking behavior.
Future work will focus on developing safe migration patterns and shadow-write strategies to accommodate live schema evolution without interrupting data flow.

## What a potential schema would look like

we're going to use a practice schema as if you're application tracked boats.

```rust

#[derive(BenSchema)]
#[bschema(
    table = "vessel_ping",
    time_to_archive = "180d",
    archive_target = "s3://ben-archives/vessel_watch/vessel_ping",
    archive_format = "Parquet"
)]
pub struct VesselPing {
    // --- Core identity fields ---
    #[bschema(key)]
    vessel_id: String,                 // IMO or MMSI number

    #[bschema(key)]
    ts_wall: u64,                      // Unix wall-clock timestamp (ms)

    #[bschema(key)]
    lat: f64,                          // Latitude
    #[bschema(key)]
    lon: f64,                          // Longitude

    // --- Motion + behavior ---
    #[bschema(threshold = ">25", warn = ">20", unit = "knots", importance = "critical")]
    speed_knots: f64,                  // Vessel speed

    #[bschema(threshold = ">30", warn = ">25", unit = "degrees", importance = "warning")]
    heading_change_deg: f64,           // Sudden heading shift, may indicate evasive maneuver

    #[bschema(unit = "nautical_miles")]
    distance_from_port: f64,           // Distance to nearest registered port

    // --- Environmental context ---
    sea_surface_temp_c: f64,           // Optional sensor data
    chlorophyll_index: f64,            // Proxy for fishing zone productivity

    // --- Categorical maps ---
    #[bschema(tags)]
    flags: HashMap<String, String>,    // E.g. { "flag_state": "PAN", "gear_type": "trawler" }

    // --- Metadata ---
    #[bschema(ben_meta)]
    meta: HashMap<String, String>,     // Optional ephemeral tags (satellite ID, batch source, etc.)
}
```

Parts:
```rust
//actual derive macro
#[derive(BenSchema)]
/*
This is the metadata which creates the table, in this case "vessel_ping".
time_to_archive is the amount of time this data stays in the ben_silo.
archive target is where the archive goes, and archive_format is the format used.
*/
#[bschema(
    table = "vessel_ping",
    time_to_archive = "180d",
    archive_target = "s3://ben-archives/vessel_watch/vessel_ping",
    archive_format = "Parquet"
)]
```
Schemas are strict lest ben become a JSON junkdrawer or elastic search with extra steps. Only structs can be 
```#[derive(BenSchema)] ```



```rust
    #[bschema(key)]
    vessel_id: String,                

    #[bschema(key)]
    ts_wall: u64,                     

    #[bschema(key)]
    lat: f64,                          
    #[bschema(key)]
    lon: f64,    
```
Denotes keys for the table

```rust
#[bschema(threshold = ">25", warn = ">20", unit = "knots", importance = "critical")]
speed_knots: f64,                  

#[bschema(threshold = ">30", warn = ">25", unit = "degrees", importance = "warning")]
heading_change_deg: f64,   
```
We encourage thresholds as it will help ben establish better baselines. These are example thresholds.


```rust
    #[bschema(tags)]
    flags: HashMap<String, String>, 

    #[bschema(ben_meta)]
    meta: HashMap<String, String>,  
```
ben allows hashmaps, though with constraints. Having huge hashmaps in the analytics db is a bad idea.


This sums up the current work on the sdk.

### Last note
I would like to note the timeline I'm on. Unfortunately, I am unable to dedicate as much time to this as I hoped, at least until next december as I'm in school taking 15 credits while designing and doing this work. Candidly I'm going for graphic design to learn data visualization for the project. Only three more semesters. Also, all of this is self learning. I during this next sumer I will be focused on getting my dashboard up with running data that shows an idea of what I mean to accomplish

Thanks for reading,
Alex