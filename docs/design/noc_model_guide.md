# NoC C++ Behavior Model

## User Guide

Version 1.0

---

## Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Feature Summary](#feature-summary)
- [Flit Format](#flit-format)
- [SystemVerilog Integration](#systemverilog-integration)
- [NoC Model API Code](#noc-model-api-code)
- [Transaction Flow](#transaction-flow)
- [Network Initialisation and Configuration](#network-initialisation-and-configuration)
- [Auxiliary Functions](#auxiliary-functions)
- [Internal Memory Access](#internal-memory-access)
- [Structure of a User Program](#structure-of-a-user-program)
- [Summary of API Functions](#summary-of-api-functions)
- [C++ API Class](#c-api-class)
- [Further Internal Architecture](#further-internal-architecture)
- [Hot-Swap and Co-Simulation](#hot-swap-and-co-simulation)
- [Traffic Monitoring and Debug Output](#traffic-monitoring-and-debug-output)
- [Simulation Scope and Limitations](#simulation-scope-and-limitations)
- [Test Environment](#test-environment)
- [Build and Installation](#build-and-installation)

---

## Introduction

The NoC (Network-on-Chip) C++ Behavior Model is a cycle-accurate simulation platform for a 2D mesh interconnect fabric. It serves two primary purposes: pre-silicon performance evaluation through high-speed parameter sweeping, and provision of a golden reference for RTL co-simulation verification.

The model arose from the need to evaluate NoC performance characteristics — latency, throughput, buffer occupancy — long before RTL implementation is available, and then to reuse the same model as a bit-exact, cycle-accurate reference against which RTL outputs can be compared. This dual role drives the fundamental architecture: the same input patterns (configuration, memory initialisation, and traffic sequences) are fed to both the C++ model and the RTL simulation, and the C++ outputs serve as the golden standard for comparison.

At its core, the model implements a configurable mesh of uniform routers connected by network interfaces (NIs). Each node comprises a Processing Element (PE) with DMA and local DRAM, a Network Interface (NMU + NSU), and a Router. External AXI masters and slaves communicate through the NIs, which perform AXI-to-flit protocol conversion. The model is written in C++17 and structured in four layers — User Code, NocSystem public API, internal components, and a Co-Simulation Bridge — with a hot-swap boundary that allows any C++ router or NI to be replaced with an RTL proxy via DPI-C.

---

## Prerequisites

The following are required to build and run the NoC C++ model:

- A C++17 compliant compiler (GCC 9+, Clang 10+, or MSVC 2019+)
- CMake 3.16 or later
- GoogleTest (fetched automatically via CMake if not installed)
- Python 3.8+ (for batch scripting and parameter sweep automation)

For RTL co-simulation, additionally:

- A SystemVerilog simulator with DPI-C support (e.g., Verilator, VCS, Questa)
- The RTL NoC modules (SystemVerilog source)

The model has no dependency on any external NoC library. It is a standalone implementation built from the ground up.

---

## Feature Summary

The model supports the following features:

- **2D Mesh Topology** — configurable from 2×2 to 16×16 (up to 256 nodes)
- **Uniform Router** — all routers identical in structure, no edge/compute distinction
- **408-bit Fixed Flit** — 56-bit header + 352-bit payload, carrying all five AXI channels (AW, W, AR, B, R) in a unified format
- **XY Deterministic Routing** — X-first then Y, deadlock-free by construction
- **Wormhole Switching** — head flit reserves the path, `last` bit releases it
- **Dual Flow Control** — Valid/Ready mode and Credit-Based mode, selected at compile time via template parameter
- **Req/Rsp Physical Separation** — independent request and response networks, eliminating protocol-level deadlock
- **QoS-Aware Arbitration** — 16-level priority with round-robin tie-breaking; pluggable allocator (RoundRobin, iSLIP, QoS-Aware RR)
- **SECDED ECC** — end-to-end data integrity (NMU generates, router passes through, NSU checks)
- **Multicast** — rectangle bounding box (RCR) multicast with in-network B response reduction
- **Reorder Buffer** — per-NMU RoB for out-of-order response support
- **Configurable Channel Delay** — link propagation modelled as N-cycle pipeline
- **Hot-Swap Interface** — individual routers or NIs replaceable with RTL proxies via DPI-C
- **Golden Verification** — automated byte-exact memory comparison and cycle-accurate response matching
- **Deadlock Detection** — configurable threshold for forward progress monitoring

---

## Flit Format

All data in the NoC is transported as 408-bit flits — a fixed-width design that eliminates configurable widths and simplifies both RTL implementation and verification. Every flit consists of a 56-bit header and a 352-bit payload. All five AXI channels (AW, W, AR, B, R) share the same flit width; shorter payloads are zero-padded to 352 bits.

### Header (56 bits)

The header is common to all flit types. Two modes exist: Valid/Ready (No-VC) and Credit-Based (With-VC), differing only in bits [55:50].

```
 55    50 49              34 33 32 31    27  26   25  24 23 22    15 14     7  6  4  3  0
┌───────┬─────────────────┬─────┬──────┬────┬────┬─────┬────────┬────────┬─────┬──────┐
│vc/rsvd│ multicast_mask   │comm │ rob  │rob │last│port │ dst_id │ src_id │ axi │ qos  │
│  6b   │      16b         │type │idx 5b│req │ 1b │id 2b│   8b   │   8b   │ch 3b│  4b  │
│       │{xmin,xmax,       │ 2b  │      │ 1b │    │     │{y,x}   │{y,x}   │     │      │
│       │ ymin,ymax}       │     │      │    │    │     │        │        │     │      │
└───────┴─────────────────┴─────┴──────┴────┴────┴─────┴────────┴────────┴─────┴──────┘
```

| Bit Range | Field | Width | Description |
|-----------|-------|-------|-------------|
| [3:0] | `qos` | 4 | QoS priority (0=best effort, 15=critical) |
| [6:4] | `axi_ch` | 3 | AXI channel type (0=AW, 1=W, 2=AR, 3=B, 4=R) |
| [14:7] | `src_id` | 8 | Source node ID（[7:4]=y, [3:0]=x） |
| [22:15] | `dst_id` | 8 | Destination node ID（[7:4]=y, [3:0]=x） |
| [24:23] | `port_id` | 2 | Target LOCAL port index (0–3) |
| [25] | `last` | 1 | Last flit in packet (releases wormhole path lock) |
| [26] | `rob_req` | 1 | RoB request flag |
| [31:27] | `rob_idx` | 5 | RoB entry index (0–31) |
| [33:32] | `commtype` | 2 | 0=Unicast, 1=Multicast, 2=ParallelReduction |
| [49:34] | `multicast_mask` | 16 | RCR bounding box {x_min, x_max, y_min, y_max} |
| [52:50] | `vc_id` | 3 | Virtual Channel ID (Credit-Based mode only; rsvd in Valid/Ready mode) |
| [55:53] | `rsvd` | 3 | Reserved (Credit-Based mode); bits [55:50] all rsvd in Valid/Ready mode |

High-frequency fields (`qos`, `axi_ch`) are placed at the LSB end so that the first pipeline stage needs only a narrow slice for arbitration and channel selection. Multicast and reserved fields sit at the MSB end, leaving room for future extension without disturbing existing bit positions.

### Payload Formats

Each AXI channel maps to a payload within the 352-bit field:

| Channel | Network | Key Fields | Used Bits | Utilisation |
|---------|---------|------------|:---------:|:-----------:|
| AW | REQ | awid(8), awaddr(64), awlen(8), awsize(3), awburst(2), awcache(4), awlock(1), awprot(3), awregion(4), awuser(8) | 108 | 30.7% |
| W | REQ | wlast(1), wuser(8), wdata(256), wstrb(32), wecc(32) | 329 | 93.5% |
| AR | REQ | arid(8), araddr(64), arlen(8), arsize(3), arburst(2), arcache(4), arlock(1), arprot(3), arregion(4), aruser(8) | 108 | 30.7% |
| B | RSP | bid(8), bresp(2), buser(8), ecc_fail(1), multicast_status(2) | 21 | 6.0% |
| R | RSP | rlast(1), rid(8), rresp(2), ruser(8), rdata(256), recc(32) | 307 | 87.2% |

W and R payloads carry 32 bits of SECDED ECC (8-bit ECC per 64-bit data granule, 4 granules). ECC is generated by the source NI, passed through routers transparently, and checked by the destination NI. A 1-bit error is auto-corrected; a 2-bit error sets `ecc_fail` in the B response or reports via `rresp`.

In a typical burst write (awlen=15), the flit sequence is 1×AW + 16×W + 1×B = 18 flits. W flits account for 89% of the total, keeping overall padding waste to approximately 8%.

### Physical Link

Each router-to-router or NI-to-router connection carries four unidirectional links: Req forward, Req reverse, Rsp forward, and Rsp reverse. Each link is 410 bits wide (1 valid + 1 ready + 408 flit data), totalling 1,640 bits per router pair.

Request and response use independent physical links (dual-rail full-duplex), eliminating request-response circular dependency as the primary protocol deadlock avoidance mechanism.

For the complete bit-level specification, see [Flit Format](02_flit.md).

---

## SystemVerilog Integration

The NoC model integrates with SystemVerilog simulators through a DPI-C bridge layer. This bridge provides two levels of API: a transaction-level interface for driving the C++ model from an SV testbench, and a signal-level interface for the hot-swap co-simulation mode.

### Transaction-Level DPI-C

The transaction-level interface allows an SV testbench to create a NocSystem instance, submit AXI transactions, step the simulation, and retrieve responses. This is the simplest integration path and is used when the C++ model runs as a standalone golden reference:

```systemverilog
import "DPI-C" function chandle noc_init(string json_path);
import "DPI-C" function void    noc_destroy(chandle h);
import "DPI-C" function int     noc_submit_write(chandle h, longint addr,
                                                  input byte data[], int len);
import "DPI-C" function int     noc_submit_read(chandle h, longint addr, int len);
import "DPI-C" function void    noc_step(chandle h, int cycles);
import "DPI-C" function int     noc_get_response(chandle h, output byte buf[], int size);
import "DPI-C" function int     noc_pending_count(chandle h);
```

The `noc_init()` function reads a JSON configuration file and constructs the entire mesh. All subsequent calls use the returned handle. The testbench submits write and read transactions, steps the simulation forward by the desired number of cycles, and then polls for responses. This mirrors the flow of a C++ user program but from within the SV environment.

### Signal-Level DPI-C

The signal-level interface is used when one or more C++ components are replaced with RTL modules (the hot-swap mode). In this case, the RTL proxy within the C++ model calls DPI-C functions every cycle to exchange port-level signals with the RTL simulator:

```systemverilog
import "DPI-C" function void noc_set_port_input(chandle h, int node, int port,
                                                 int ch, bit valid,
                                                 input bit [407:0] flit,
                                                 int ready_or_credit);
import "DPI-C" function void noc_get_port_output(chandle h, int node, int port,
                                                  int ch, output bit valid,
                                                  output bit [407:0] flit,
                                                  output int ready_or_credit);
```

The `ch` parameter selects the channel: 0 for REQ, 1 for RSP. The `ready_or_credit` parameter carries a ready bit in valid/ready mode or a per-VC credit bitmask in credit mode. Each cycle, the RTL proxy sets its input signals from the RTL module's outputs and provides its output signals for the RTL module's inputs, maintaining cycle-accurate synchronisation between the C++ mesh and the RTL component under test.

### RTL Module Instantiation

The RTL side instantiates the NoC router or NI module under test, connects its ports to DPI-C wrapper signals, and calls the bridge functions at every clock edge. An example top-level for verifying a single RTL router:

```systemverilog
module noc_cosim_top;
    chandle noc_handle;
    // ... clock, reset, port signals ...

    initial begin
        noc_handle = noc_init("config.json");
        // ... reset sequence ...
    end

    always @(posedge clk) begin
        // Push RTL outputs into C++ model
        noc_set_port_input(noc_handle, NODE_ID, port, ch,
                           rtl_out_valid, rtl_out_flit, rtl_out_ready);
        // Pull C++ model outputs to drive RTL inputs
        noc_get_port_output(noc_handle, NODE_ID, port, ch,
                            rtl_in_valid, rtl_in_flit, rtl_in_ready);
        // Step C++ model
        noc_step(noc_handle, 1);
    end
endmodule
```

---

## NoC Model API Code

The NoC model's software architecture is shown below. Each block communicates through well-defined interfaces.

```
┌──────────────────────────────────────────────────────────────────────┐
│ User Code                                                              │
│   C++ test program / SV testbench / Python batch script               │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ C++ API / DPI-C
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ NocSystem Public API (noc_api.h)                                      │
│   Group A: Construction    Group B: Transaction    Group C: Simulation │
│   Group D: Completion      Group E: Metrics        Group F: Debug     │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ delegates to
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Internal Components                                                    │
│   TrafficManager / Mesh / Router / NI / Channel / Memory / Stats      │
│   ┌──────────────────── HOT-SWAP BOUNDARY ───────────────────────┐  │
│   │  CppRouter ◄── Router_Interface<Mode> ──► RouterDpiBridge     │  │
│   │  CppNI     ◄── NI_Interface<Mode>    ──► NIDpiBridge          │  │
│   └──────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ DPI-C
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Co-Sim Bridge + RTL                                                    │
│   Signal-Level DPI-C / RTL Router / RTL NI (SystemVerilog)            │
└──────────────────────────────────────────────────────────────────────┘
```

At the heart of the model is the NocSystem class, which owns the entire mesh and exposes six groups of API functions. User code interacts exclusively with this Public API. The NocSystem delegates to internal components: a TrafficManager that handles transaction lifecycle, a Mesh that owns all routers, NIs, and channels, and support components for online transaction verification (Scoreboard) and performance statistics (MetricsCollector).

The main processing loop is driven by `process_cycle()`, which executes an 8-phase pipeline for each simulation cycle (detailed Phase 4 state machine and credit timing in [Simulation Platform](08_simulation.md) §6). The phases decompose RTL's inherently parallel behaviour into a deterministic sequential ordering:

| Phase | Name | Action |
|-------|------|--------|
| 1 | Sample | Latch inputs: `in_valid && out_ready` → push to buffer |
| 2 | Clear Inputs | Prevent double-sampling |
| 3 | Update Ready | `out_ready = !buffer.full()` (combinational) |
| 4 | Route & Forward | RC → VA → SA → ST pipeline + credit return generation |
| 5 | Wire All | Channel\<T\> propagates outputs to downstream inputs |
| 6 | Clear Accepted | `out_valid && in_ready` → clear output register |
| 7 | Credit Update | Upstream credit counter incremented (Credit-Based mode; no-op for Valid/Ready) |
| 8 | NI Process | NMU/NSU AXI ↔ flit conversion |

Phases 1–4 are encapsulated in `Router_Interface::tick()`, Phase 5 is the global wiring step managed by the Mesh, Phases 6–7 are in `Router_Interface::post_wire()`, and Phase 8 is `NI_Interface::tick()`. This decomposition maps directly to RTL: sequential phases correspond to `posedge clk` behaviour, and combinational phases to `assign` logic.

---

## Transaction Flow

A transaction in the NoC model follows a well-defined path from submission to completion. The TrafficManager sits between the NocSystem API and the mesh, managing the entire lifecycle.

### Write Transaction

When the user submits a write via `submit_write()`, the TrafficManager creates a pending transaction record, registers the expected outcome in the Scoreboard for online verification, and queues it for injection. On each `tick()`, the TrafficManager checks whether the transaction's injection cycle has arrived and, if so, delivers the AXI write request (address, data, length) to the appropriate NMU.

The NMU performs address translation to determine the destination node ID, generates QoS priority, packs the AXI AW channel into a header flit, and packs each W beat into one or more data flits. SECDED ECC is appended. The resulting flit sequence enters the request network, traverses routers via XY routing with wormhole path locking, and arrives at the destination NSU.

The NSU unpacks the flits back into AXI format, verifies ECC, reassembles W burst data, and writes to local memory. It then generates a B response flit, which travels back through the response network to the originating NMU. The NMU's reorder buffer (RoB) matches the response to the outstanding request and delivers the completion to the TrafficManager.

### Read Transaction

Read transactions follow a similar path. The NMU packs an AR flit which traverses the request network to the destination NSU. The NSU reads local memory and generates one or more R response flits (one per beat for burst reads). These traverse the response network back to the NMU, where the RoB reassembles the data and signals completion.

### Multicast Write

Multicast writes use the RCR (Rectangle Covering Route) mechanism. The NMU sets a multicast bitmask in the flit header defining a rectangular bounding box of destination nodes. Routers within the bounding box replicate the flit to the appropriate output ports using purely combinational logic (8 comparators per router). B responses from all destinations are reduced in-network: the response router merges multiple B flits into a single response before forwarding upstream. A timeout mechanism (configurable via `REDUCTION_TIMEOUT`) prevents indefinite waiting for responses.

### Flit Mapping

All data flows use a uniform 408-bit flit (56-bit header + 352-bit payload). The mapping from AXI transactions to flits is:

| AXI Transaction | Request Flits | Response Flits |
|-----------------|:-------------:|:--------------:|
| Write (awlen=0) | 1 AW + 1 W = 2 | 1 B |
| Write (awlen=N) | 1 AW + (N+1) W | 1 B |
| Read (arlen=0) | 1 AR | 1 R |
| Read (arlen=N) | 1 AR | (N+1) R |

---

## Network Initialisation and Configuration

The model works through a single configuration object, NocConfig, which serves as the single source of truth for all system parameters. A JSON file is loaded at startup and the resulting NocConfig is passed to the NocSystem constructor.

### NocConfig Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Topology** | | |
| `MESH_COLS` | 4 | Number of mesh columns |
| `MESH_ROWS` | 4 | Number of mesh rows |
| `DEFAULT_LOCAL_PORTS` | 1 | Default LOCAL ports per router (0–4) |
| `NODE_LOCAL_PORTS` | `{}` | Per-node override (node_id → port count) |
| **Buffer** | | |
| `INPUT_BUFFER_DEPTH` | 4 | Router input buffer depth (flits) |
| `OUTPUT_BUFFER_DEPTH` | 2 | Router output buffer depth (0 = wire-through) |
| `NMU_BUFFER_DEPTH` | 2 | NMU injection buffer depth |
| **Flow Control** | | |
| `FLOW_CONTROL` | VALID_READY | Flow control mode (factory selects template) |
| `NUM_VC` | 1 | Number of virtual channels |
| **Pipeline Delay** | | |
| `ROUTING_DELAY` | 0 | Route computation extra cycles |
| `VC_ALLOC_DELAY` | 0 | VC allocation extra cycles |
| `SW_ALLOC_DELAY` | 0 | Switch allocation extra cycles |
| `CREDIT_DELAY` | 1 | Credit return latency (cycles) |
| `CHANNEL_DELAY` | 1 | Link propagation latency (cycles) |
| **Allocator** | | |
| `VC_ALLOCATOR` | `"round_robin"` | VC allocator strategy |
| `SW_ALLOCATOR` | `"qos_aware_rr"` | Switch allocator strategy |
| **Traffic** | | |
| `MAX_OUTSTANDING` | 8 | Maximum outstanding transactions |
| `MAX_BURST_LEN` | 16 | Maximum AXI burst length |
| **NI** | | |
| `ROB_ENTRIES` | 32 | Reorder buffer entries per NI |
| **Memory** | | |
| `MEMORY_READ_LATENCY` | 0 | Local memory read latency (0 = ideal) |
| `MEMORY_WRITE_LATENCY` | 0 | Local memory write latency (0 = ideal) |
| **Safety** | | |
| `REDUCTION_TIMEOUT` | 1000 | Multicast reduction timeout cycles |
| `CREDIT_TIMEOUT` | 10000 | Credit starvation detection cycles (0 = disabled) |
| `MULTICAST_TIMEOUT` | 10000 | Multicast handshake timeout cycles (0 = disabled) |
| `DEADLOCK_THRESHOLD` | 1000 | Forward progress timeout cycles |

### Fixed Design Parameters

The following parameters are fixed at design time and cannot be changed during simulation:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `FLIT_WIDTH` | 408 bits | Header 56 + Payload 352 |
| `AXI_DATA_WIDTH` | 256 bits | AXI data bus width |
| `AXI_ADDR_WIDTH` | 64 bits | AXI address width |
| `NODE_ID_WIDTH` | 8 bits | [7:4]=y, [3:0]=x |
| `QOS_WIDTH` | 4 bits | 16 QoS levels |
| `ECC_WIDTH` | 32 bits | SECDED |

### JSON Configuration Example

```json
{
  "topology": { "mesh_cols": 5, "mesh_rows": 4, "default_local_ports": 1 },
  "router":   { "input_buffer_depth": 4, "num_vc": 1, "flow_control": "valid_ready" },
  "ni":       { "nmu_buffer_depth": 2, "rob_entries": 32 },
  "pipeline": { "routing_delay": 0, "channel_delay": 1 }
}
```

### Memory Initialisation

Pre-loading of host memory and per-node local memory uses `$readmemh`-compatible `.hex` files:

```
@00000000 AB CD EF 01 02 03 04 05 ...
```

Each node has its own `.hex` file (optional; uninitialised nodes default to zero). The same files are consumed by both the C++ model (via `load_memory()`) and the RTL simulation (via `$readmemh`), ensuring identical starting conditions.

### Traffic Definition

The traffic pattern is defined in a JSON file specifying a sequence of AXI transactions:

| Field | Description |
|-------|-------------|
| `id` | Unique transaction identifier |
| `type` | `WRITE` / `READ` / `MULTICAST_WRITE` |
| `src_node` | Source NMU node ID |
| `dst_addr` | 64-bit AXI address (`[39:32]` = node_id, `[31:0]` = local_addr) |
| `dst_nodes` | Multicast destination list (MULTICAST_WRITE only) |
| `data_file` | Path to `.hex` file containing write data |
| `data_size` | Transfer size in bytes |
| `axi_id` | AXI transaction ID |
| `burst_len` | AXI burst length (beats) |
| `inject_cycle` | Absolute cycle number or `"after:<txn_id>"` |

The TrafficManager loads this file and schedules injection according to the `inject_cycle` field. This allows both absolute timing and dependency-based sequencing of transactions.

### Injection Modes

The TrafficManager supports two injection modes, applicable to both traffic and memory initialisation:

| Mode | Description |
|------|-------------|
| `HOST_DMA` | All transactions are injected from a designated gateway node's Host DMA NI (`port_id=1`) |
| `DIRECT_PE` | Transactions are distributed to their respective source node's PE NI (`port_id=0`) |

The injection mode is set via `set_injection_mode()` and the gateway node is configured with `set_gateway_node()`. In `HOST_DMA` mode, the gateway node's router must have at least 2 LOCAL ports.

---

## Auxiliary Functions

### Completion Handling

The `is_complete()` function tests whether a given transaction (identified by its `TxnHandle`) has received its response. If the transaction is still pending, the function returns false. The companion `all_complete()` checks all submitted transactions at once.

For non-blocking designs, `set_completion_callback()` registers a user function that is invoked whenever a transaction completes. The callback receives the transaction handle and status, allowing the user code to process completions asynchronously rather than polling.

The transaction status follows a state machine:

```
PENDING → INJECTING → WAITING_RESPONSE → COMPLETE / ERROR / TIMEOUT
```

Multiple transactions can be submitted before waiting for any completions. If, at a later point, `is_complete()` is called and the transaction has already completed, it simply returns true immediately without blocking. To wait for all outstanding transactions at once, `run_until_idle()` can be used, which advances the simulation until the network is quiescent and all transactions are complete.

### Deadlock Detection

The `run_until_idle()` and `run_all()` functions include built-in deadlock detection. Forward progress is defined as at least one router input buffer experiencing a `pop` operation. If no forward progress occurs for `DEADLOCK_THRESHOLD` consecutive cycles (default 1000), the simulation emits a warning and aborts. This threshold is configurable via NocConfig.

### Cycle Counter

The `current_cycle()` function returns the current simulation cycle count, useful for timing measurements and debug output correlation.

---

## Internal Memory Access

Each node in the NoC mesh has its own local memory, modelled by the LocalMemory class. Additionally, a shared HostMemory represents the external host's memory space. Both can be accessed from user code through the NocSystem API.

To write to a node's local memory, `load_local_memory()` is used, which takes a node ID, a byte-aligned address, a pointer to data, and a byte length. To read back memory contents, `dump_local_memory()` retrieves the contents into a user-provided buffer. For the host memory, `load_host_memory()` and corresponding read functions are provided.

Memory can also be initialised in bulk from `.hex` files using `load_memory()`, which scans a directory for per-node hex files and loads each into the corresponding node's local memory.

The memory model uses a flat address space per node. Reads from uninitialised locations return zero. The model does not enforce 4K boundary alignment for individual transactions — this is handled by the NI's burst splitting logic.

The Scoreboard records expected outcomes at the point of submission (before data enters the network). When a response arrives, it immediately compares actual vs expected per-transaction. `get_summary()` returns the overall PASS/FAIL status. This online approach detects mismatches as soon as they occur, rather than waiting until simulation ends.

| Function | Description |
|----------|-------------|
| `load_host_memory(addr, data, len)` | Write to host memory |
| `load_local_memory(node_id, addr, data, len)` | Write to a node's local memory |
| `dump_local_memory(node_id, addr, buf, len)` | Read from a node's local memory |
| `load_memory(hex_dir)` | Bulk load from `.hex` files |

---

## Structure of a User Program

There are many ways that a user program could be constructed to utilise the NoC model, but there are some common components that would feature in any implementation, which are discussed now. Below is shown an example outline of a basic user program:

```cpp
#include "noc/noc_api.h"
#include <cstdio>
#include <cstdlib>
#include <vector>

static unsigned error_count = 0;

static void on_complete(noc::TxnHandle h, noc::TxnStatus status) {
    if (status == noc::TxnStatus::ERROR || status == noc::TxnStatus::TIMEOUT) {
        fprintf(stderr, "Transaction %u FAILED (status=%d)\n", h, static_cast<int>(status));
        error_count++;
    }
}

int main() {
    // Load configuration
    auto config = noc::NocSystem::load_config("patterns/config.json");
    noc::NocSystem system(config);

    // Load initial memory contents
    system.load_memory("patterns/");

    // Register completion callback for error tracking
    system.set_completion_callback(on_complete);

    // Prepare write data: 32 bytes
    std::vector<uint8_t> wdata(32, 0xA5);

    // Submit a write to node (1,0), local address 0x0000
    // Address encoding: [39:32]=node_id, [31:0]=local_addr
    auto h1 = system.submit_write(0x01'0000'0000ULL, wdata.data(), 32, /*axi_id=*/0);

    // Submit a read-back from the same address
    auto h2 = system.submit_read(0x01'0000'0000ULL, 32, /*axi_id=*/1);

    // Run until all transactions complete (max 10000 cycles)
    system.run_until_idle(10000);

    // Check individual transaction status
    if (system.get_status(h1) != noc::TxnStatus::COMPLETE) {
        fprintf(stderr, "Write did not complete\n");
        return 1;
    }
    if (system.get_status(h2) != noc::TxnStatus::COMPLETE) {
        fprintf(stderr, "Read did not complete\n");
        return 1;
    }

    // Retrieve and verify read data
    auto rdata = system.get_read_data(h2);
    if (rdata != wdata) {
        fprintf(stderr, "Data mismatch!\n");
        return 1;
    }

    // Run golden verification (memory state + response log)
    auto report = system.verify();
    printf("Verification: %s (%u mismatches)\n",
           report.passed ? "PASS" : "FAIL", report.mismatch_count);

    // Generate golden output files for RTL comparison
    system.generate_golden("golden/");

    printf("Simulation completed at cycle %lu, errors=%u\n",
           system.current_cycle(), error_count);
    return error_count > 0 ? 1 : 0;
}
```

The above code assumes a default 4×4 mesh. The call to `load_config()` reads the JSON configuration and constructs a NocConfig object. The NocSystem constructor builds the entire mesh — routers, NIs, channels, and memories — according to this configuration.

The completion callback tracks errors as they occur, while `get_status()` checks individual transactions after simulation. The destination node is encoded in bits `[39:32]` of the address — in this case node `0x01` (coordinates (1,0)). After the read completes, the returned data is compared against the original write data. Finally, `verify()` runs the full golden verification and `generate_golden()` produces output files for RTL comparison.

The `run_until_idle()` call advances the simulation cycle by cycle until all transactions have completed or the maximum cycle count is reached. The model's deadlock detection will abort early if the network stalls.

At its simplest, a user program is just calls to `submit_write()`, `submit_read()`, and `run_all()`. The model handles all the complexity of flit packing, routing, arbitration, and response matching internally. For more control, the user can call `process_cycle()` individually to step through the simulation one cycle at a time, inspecting internal state along the way.

For traffic-file-driven simulation, the flow is even simpler:

```cpp
auto config = noc::NocSystem::load_config("patterns/config.json");
noc::NocSystem system(config);
system.load_memory("patterns/");
system.load_traffic("patterns/traffic.json");
system.run_all();
system.generate_golden("golden/");
```

The `load_traffic()` function reads the traffic JSON and schedules all transactions internally. `run_all()` then executes the simulation to completion. This is the typical flow for batch regression runs and parameter sweeps.

---

## Summary of API Functions

### Group A: Construction & Configuration

```cpp
static NocConfig   load_config(const std::string& json_path);
                    NocSystem(const NocConfig& config);
void               load_memory(const std::string& hex_dir);
void               load_traffic(const std::string& json_path);
void               load_host_memory(uint64_t addr, const uint8_t* data, size_t len);
void               load_local_memory(uint8_t node_id, uint64_t addr,
                                     const uint8_t* data, size_t len);
void               dump_local_memory(uint8_t node_id, uint64_t addr,
                                     uint8_t* buf, size_t len);
```

### Group B: Transaction Submission

```cpp
TxnHandle          submit_write(uint64_t addr, const uint8_t* data,
                                size_t len, uint8_t axi_id);
TxnHandle          submit_read(uint64_t addr, size_t len, uint8_t axi_id);
TxnHandle          submit_multicast_write(uint64_t addr, const uint8_t* data,
                                          size_t len,
                                          const std::vector<uint8_t>& dst_nodes);
```

### Group C: Simulation Control

```cpp
void               process_cycle();
void               run(uint32_t cycles);
void               run_until_idle(uint32_t max_cycles);
void               run_all();
uint64_t           current_cycle();
```

### Group D: Completion & Response

```cpp
bool               is_complete(TxnHandle h);
bool               all_complete();
TxnStatus          get_status(TxnHandle h);
std::vector<uint8_t> get_read_data(TxnHandle h);
void               set_completion_callback(CompletionCallback cb);
```

### Group E: Metrics & Verification

```cpp
MetricsCollector&  get_metrics();
VerificationReport verify();
```

### Group F: Debug & Output

```cpp
const Router&      get_router(Coord coord);
const NI&          get_ni(Coord coord);
void               dump_state(std::ostream& os);
void               generate_golden(const std::string& output_dir);
void               dump_response_log(const std::string& path);
```

### TrafficManager Functions

```cpp
void               set_injection_mode(InjectionMode mode);
void               set_gateway_node(uint8_t node_id, uint8_t port_id);
TxnHandle          submit_write(uint64_t addr, const uint8_t* data,
                                size_t len, uint8_t axi_id);
TxnHandle          submit_read(uint64_t addr, size_t len, uint8_t axi_id);
void               tick(uint64_t current_cycle);
bool               is_complete(TxnHandle h);
TxnStatus          get_status(TxnHandle h);
std::vector<uint8_t> get_read_data(TxnHandle h);
```

### DPI-C Transaction-Level Functions

```cpp
void*              noc_init(const char* json_path);
void               noc_destroy(void* handle);
int                noc_submit_write(void* h, uint64_t addr,
                                    const uint8_t* data, int len);
int                noc_submit_read(void* h, uint64_t addr, int len);
void               noc_step(void* h, int cycles);
int                noc_get_response(void* h, uint8_t* buf, int size);
int                noc_pending_count(void* h);
```

### DPI-C Signal-Level Functions

```cpp
void               noc_set_port_input(void* h, int node, int port, int ch,
                                      int valid, const uint8_t* flit,
                                      int ready_or_credit);
void               noc_get_port_output(void* h, int node, int port, int ch,
                                       int* valid, uint8_t* flit,
                                       int* ready_or_credit);
int                noc_get_num_ports(void* h, int node);
```

---

## C++ API Class

In addition to the C-style DPI-C API described in the previous sections, the primary interface to the model is the `noc::NocSystem` C++ class defined in `include/noc/noc_api.h`. The class encapsulates all six API groups as member functions.

The constructor takes a `NocConfig` object (typically obtained from `load_config()`) and builds the complete mesh, including all routers, NIs, channels, and memory instances. The mesh topology, buffer depths, flow control mode, and all other parameters are determined at construction time from the configuration.

The flow control mode is resolved at compile time through a factory pattern. The user-facing `NocSystem` is a type-erased wrapper around a templated `NocSystemImpl<Mode>`:

```
NocSystemBase (type-erased)
    ├── NocSystemImpl<VALID_READY>
    └── NocSystemImpl<CREDIT>

create_noc_system(config) → switch(config.flow_control) → NocSystemImpl
```

This is analogous to RTL parameter elaboration — the template parameter selects which flow control signals are instantiated, and no unused signals exist in the compiled model.

---

## Further Internal Architecture

### Router Architecture

Each router consists of two structurally symmetric sub-routers: a ReqRouter for AW/W/AR flits and a RspRouter for B/R flits. The RspRouter additionally includes in-network reduction logic for multicast B response merging.

The router pipeline within Phase 4 processes each head flit through four sub-stages, each with a configurable delay:

```
InputQueuing → RouteEvaluate → VCAllocEvaluate → SWAllocEvaluate → SwitchTraverse → OutputQueuing
```

Body and tail flits of an active wormhole path skip RC/VA/SA and proceed directly to SwitchTraverse, following the path lock established by the head flit.

The router's sub-modules and their responsibilities:

| Sub-Module | Description |
|------------|-------------|
| InputBuffer | Per-port FIFO (configurable depth) |
| BufferState | Credit tracking, separated from data storage |
| RouteCompute | XY routing + RCR multicast routing |
| Crossbar | N×N switch fabric |
| PathLock | Wormhole path locking FSM |
| Allocator | Pluggable: RoundRobin / iSLIP / QoSAwareRR |
| MulticastTracker | Multicast fork tracking |
| ReductionSync / ReductionArb | In-network B response reduction |

### Network Interface Architecture

The NI contains two units: the NMU (Network Master Unit) on the request-issuing side, and the NSU (Network Slave Unit) on the request-serving side.

**NMU pipeline (request path):**
AXI Slave → AddrTranslate → QoSGenerate → FlitPack (AW/W/AR) → ECC Generate → Injection Buffer → Network

**NMU pipeline (response path):**
Network → ECC Check → FlitUnpack (B/R) → Reorder Buffer → AXI Slave

**NSU pipeline (request path):**
Network → FlitUnpack (AW/AR/W) → Reassembly → ECC Check → ReqInfoStore → Memory Operation

**NSU pipeline (response path):**
Memory → AXI Response → FlitPack (B/R) → ECC Generate → Network

Key sub-modules:

| Sub-Module | Description |
|------------|-------------|
| InjectionBuffer | NMU → network FIFO (req/rsp each) |
| ReorderBuffer (RoB) | NMU response reordering (32 entries default) |
| ReassemblyBuffer | NSU W burst reassembly |
| ReqInfoStore | NSU stores src/qos/rob_idx for response routing |
| LocalMemory | Per-node flat memory model |

### Channel\<T\> Link Model

The Channel class models link propagation delay between components, replacing zero-cycle hard-wired connections. It is implemented as a shift register pipeline:

| CHANNEL_DELAY | Behaviour | RTL Equivalent |
|---------------|-----------|----------------|
| 0 | Zero-cycle (combinational wire) | `assign` |
| 1 | Available next cycle (default) | 1-stage pipeline register |
| N>1 | Available after N cycles | N-stage pipeline register |

During Phase 5, the global `wire_all()` step processes all channels in strict order: ReadInputs → WriteOutputs → Deliver → Send. This ordering is critical — if Send were to execute before ReadInputs, a delay-1 channel would behave as delay-0.

The mesh maintains two sets of channels: router-to-router channels for mesh links (N/S/E/W), and NI-to-router channels for LOCAL port connections. Each channel carries source/destination component references, port indices, and channel type (REQ or RSP).

### Allocator

The allocator is an abstract base class with a single `allocate(requests, grants)` method. Three implementations are provided:

| Implementation | Description |
|----------------|-------------|
| RoundRobin | Basic round-robin arbitration |
| iSLIP | Iterative round-robin matching (multi-iteration) |
| QoSAwareRR | QoS priority with round-robin tie-breaking (default) |

The allocator strategy is selected by the NocConfig string (`vc_allocator`, `sw_allocator`) and instantiated by the factory at construction time.

### Buffer / BufferState Separation

The InputBuffer stores flit data (push/pop/peek/full/empty), while the BufferState tracks credit counts independently. This separation allows the credit tracking logic to operate without accessing the actual buffer data, which is important for the credit-based flow control mode. The credit invariant is:

```
upstream.credit[vc] + downstream.buffer.count(vc) + in_flight(vc) == INPUT_BUFFER_DEPTH_PER_VC
```

---

## Hot-Swap and Co-Simulation

The NoC model can be configured to replace individual C++ routers or NIs with RTL implementations for verification purposes. This is achieved through the hot-swap mechanism, which relies on the abstract `Router_Interface<Mode>` and `NI_Interface<Mode>` base classes.

### Substitution Modes

| Mode | Router | NI | Use Case |
|------|--------|----|----------|
| Pure C++ | CppRouter | CppNI | High-speed performance evaluation |
| NI RTL | CppRouter | NIDpiBridge | Verify NI RTL against C++ golden |
| Router RTL | RouterDpiBridge | CppNI | Verify Router RTL against C++ golden |
| Full RTL | RouterDpiBridge | NIDpiBridge | Full system co-simulation |

In the pure C++ mode, maximum simulation speed is achieved. As RTL components are substituted, simulation speed decreases proportionally due to the DPI-C overhead and RTL simulator step time, but the same input patterns and expected outputs apply.

### Configuring Substitution

The substitution targets are specified in the JSON configuration file under a `cosim` section. The following example replaces the router at node (2,1) with an RTL proxy:

```json
{
  "topology": { "mesh_cols": 4, "mesh_rows": 4 },
  "router":   { "input_buffer_depth": 4, "flow_control": "credit" },
  "cosim": {
    "rtl_routers": [ {"node_id": "0x12", "comment": "node (2,1)"} ],
    "rtl_nis":     []
  }
}
```

The `node_id` uses the same encoding as the flit header（[7:4]=y, [3:0]=x）. Multiple nodes can be listed to replace several routers or NIs simultaneously. In user code, the factory function reads this section and instantiates `RouterDpiBridge` or `NIDpiBridge` for the designated nodes, while all other nodes use the C++ implementations:

```cpp
auto config = noc::NocSystem::load_config("cosim_config.json");
// Factory automatically creates RouterDpiBridge for node 0x12
noc::NocSystem system(config);

system.load_memory("patterns/");
system.load_traffic("patterns/traffic.json");
system.run_all();

// Golden outputs remain identical regardless of substitution
system.generate_golden("golden/");
```

The user program itself does not change — the substitution is entirely configuration-driven. The same traffic patterns produce the same golden outputs regardless of which nodes use C++ or RTL implementations.

### Hot-Swap Constraints

Substitution can only occur when the target component's channels are empty (no in-flight flits). The model does not support migrating C++ internal state to RTL register state or vice versa; thus, hot-swap must happen during network quiescence or after draining the relevant channels. In practice, substitution is configured before the simulation begins and remains fixed for the duration of the run.

### Interface Contract

The interface is the contract. Both CppRouter and RouterDpiBridge implement the same `Router_Interface<Mode>`, and the Mesh wiring loop connects components solely through this interface. The flow control mode is a compile-time template parameter, ensuring that valid/ready mode carries no credit signals and credit mode carries no ready signal — no unused wires exist.

NI-to-router connections use the same interface and wiring as router-to-router connections, so the hot-swap mechanism applies uniformly to both.

### Non-Replaceable Components

| Component | Reason |
|-----------|--------|
| TrafficManager | Central coordinator, no RTL equivalent |
| Mesh topology | Fixed wiring, parameterised by config |
| Channel\<T\> | Link model, no RTL equivalent |
| Scoreboard / MetricsCollector | Verification/stats logic, C++ only |
| HostMemory / LocalMemory | Behavioural model, no RTL equivalent |

---

## Traffic Monitoring and Debug Output

The NoC model provides several mechanisms for monitoring traffic and debugging simulation behaviour.

### Output Patterns

The model generates four types of output:

| Output | Format | Purpose | Golden Compare |
|--------|--------|---------|:--------------:|
| Memory State | `.hex` | Final memory contents of all nodes | byte-exact |
| Response Log | `.json` + `.hex` | Per-transaction AXI response + read data | cycle-accurate* |
| Cycle Trace | `.vcd` / `.json` | Per-cycle router/NI/channel state | debug only |
| Statistics | `.json` | Latency, throughput, buffer occupancy | derived |

\* Cycle-accurate match requires that NocConfig pipeline delay parameters are correctly calibrated against RTL pipeline depths.

### Golden Verification Flow

The same set of input patterns is fed to both the C++ model and the RTL simulation. The C++ outputs serve as the golden reference:

```
                  ┌─────────────┐
                  │   Inputs    │
                  │ config.json │
                  │ memory/*.hex│
                  │ traffic.json│
                  └──────┬──────┘
              ┌──────────┴──────────┐
              ▼                     ▼
    ┌──────────────────┐  ┌──────────────────┐
    │  C++ NoC Model   │  │  RTL Simulation  │
    │  (golden ref)    │  │  (DUT)           │
    └────────┬─────────┘  └────────┬─────────┘
             ▼                     ▼
    ┌──────────────────┐  ┌──────────────────┐
    │  Golden Output   │  │  Actual Output   │
    └────────┬─────────┘  └────────┬─────────┘
             └──────────┬──────────┘
                        ▼
                  ┌───────────┐
                  │ Comparator│
                  │           │
                  │ Memory:   │
                  │  byte-exact│
                  │ Response: │
                  │  cycle-   │
                  │  accurate │
                  └─────┬─────┘
                   PASS / FAIL
```

### MetricsCollector

The MetricsCollector records per-node and per-transaction performance metrics throughout the simulation:

| Function | Description |
|----------|-------------|
| `record_injection(node, cycle)` | Record flit injection timestamp |
| `record_ejection(node, cycle)` | Record flit ejection timestamp |
| `get_latency_stats()` | Min/max/avg latency statistics |
| `get_throughput_stats()` | Throughput statistics |
| `get_buffer_stats()` | Buffer occupancy statistics |
| `dump_json(path)` | Export statistics as JSON |

### State Dump

The `dump_state()` function outputs the complete internal state of every router, NI, channel, and memory to an output stream. This is useful for post-mortem debugging when a simulation fails or produces unexpected results. Individual components can also be inspected via `get_router(coord)` and `get_ni(coord)`, which return read-only references.

---

## Simulation Scope and Limitations

### What Is Modelled

| Feature | Modelled | Notes |
|---------|:--------:|-------|
| Cycle-accurate timing | Y | 8-phase pipeline with configurable delays |
| Wormhole switching | Y | Path lock/release FSM per head flit |
| Credit-based flow control | Y | Per-VC credit counters (Credit-Based mode) |
| Valid/Ready handshake | Y | AXI-style backpressure (Valid/Ready mode) |
| AXI protocol (AW/W/AR/B/R) | Y | Burst, reorder, interleaving |
| QoS-aware arbitration | Y | 16-level priority + round-robin tie-break |
| ECC generate/check | Y | SECDED behavioural model |
| Multicast + in-network reduction | Y | RCR bounding box + B response merging |
| Configurable channel delay | Y | Channel\<T\> N-cycle pipeline |
| Deadlock detection | Y | Configurable forward progress timeout |

### What Is NOT Modelled

| Feature | Notes |
|---------|-------|
| Clock domain crossing | Global synchronous clock assumed |
| Power estimation | No power modelling or power gating |
| Physical wire delay | Abstracted into `CHANNEL_DELAY` parameter |
| Reset sequence | Model initialises to ready state directly |
| FF setup/hold / Metastability | Synchronous design assumption |
| AXI slave stall | AXI slave always ready (no back-pressure from memory) |
| AXI4-Stream / Atomic ops | Only AXI4 five-channel protocol supported |

### Timing Abstraction

The model decomposes RTL's parallel behaviour into 8 sequential phases. Combinational logic completes instantaneously within a phase. Wire propagation is abstracted as `CHANNEL_DELAY` cycles. Pipeline registers map to channel delay plus output buffers.

### Known Caveats

1. **Pipeline delay calibration** — The Phase 4 per-flit state machine delays (`ROUTING_DELAY`, `VC_ALLOC_DELAY`, `SW_ALLOC_DELAY`) must be calibrated against RTL pipeline depths for cycle-accurate matching. Incorrect calibration produces functionally correct but timing-shifted results.
2. **Memory latency** — Configurable via `MEMORY_READ_LATENCY` / `MEMORY_WRITE_LATENCY` (default 0 = ideal). Non-zero values add cycles to NSU memory access but do not model DRAM refresh or bank conflicts.
3. **Header integrity** — ECC protects only payload data (wdata/rdata). The 56-bit header and payload metadata (wstrb, axi_id, etc.) rely on physical link layer integrity. A `dst_id` bit flip will cause misrouting.

---

## Test Environment

The model includes a test environment that exercises the NoC mesh using GoogleTest. Tests are organised in three levels:

### Unit Tests

Each internal component has dedicated unit tests:

| Component | Test Coverage |
|-----------|---------------|
| InputBuffer | Push/pop, full/empty, overflow |
| BufferState | Credit consume/release, invariant |
| RouteCompute | XY routing correctness, boundary cases |
| Crossbar | N×N switching, contention |
| PathLock | Wormhole lock/release FSM |
| Allocator | Fairness, priority, starvation-free |
| NMU | AXI→flit packing, ECC generation |
| NSU | Flit→AXI unpacking, ECC checking |
| Channel\<T\> | Delay pipeline, ordering |
| RoB | Reorder, timeout, overflow |
| LocalMemory | Read/write, hex load/dump |
| TrafficManager | Injection scheduling, completion |

### Integration Tests

Integration tests exercise end-to-end transaction flows through a small mesh:

| Test | Description |
|------|-------------|
| Single write/read | Basic round-trip through 2×2 mesh |
| Burst transfer | Multi-beat AXI burst, N flits |
| Multi-hop | 4×4 mesh, corner-to-corner |
| Backpressure | Buffer full → flow control |
| Credit flow | Credit-Based mode credit exchange |
| Multicast | RCR multicast + B reduction |
| QoS priority | High-priority preemption |
| Deadlock detection | Timeout and abort |
| Multiple outstanding | N transactions in flight |
| Gateway injection | HOST_DMA mode |

### Co-Simulation Tests

Co-simulation tests replace one C++ component with its RTL counterpart and verify that the mixed simulation produces identical outputs to the pure C++ run.

---

## Build and Installation

The model uses CMake as its build system. To build:

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --parallel
```

To run the test suite:

```bash
ctest --output-on-failure
```

### Directory Structure

```
noc_c_model/
├── include/noc/         Public headers (noc_api.h, types.h, config.h)
├── src/core/            Router, NI, Flit, Buffer, Arbiter, Routing, Mesh
├── src/memory/          LocalMemory, HostMemory
├── src/system/          NocSystem, Scoreboard
├── src/stats/           MetricsCollector
├── src/bridge/          DPI-C and VPI simulator bridge
├── rtl/                 Minimal RTL harness (SystemVerilog)
├── tests/               Unit, integration, co-sim tests (GoogleTest)
├── examples/patterns/   Example config.json, traffic.json, memory .hex
└── docs/                Design documents and reference material
```

### Namespace

All public symbols are in the `noc::` namespace. Internal implementation details are in `noc::detail::`.

### CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `NOC_BUILD_TESTS` | ON | Build GoogleTest test suite |
| `NOC_BUILD_COSIM` | OFF | Build DPI-C bridge for co-simulation |
| `NOC_FLOW_CONTROL` | VALID_READY | Default flow control mode |

The model is available in the project repository. All dependencies (GoogleTest) are fetched automatically via CMake's FetchContent mechanism.
