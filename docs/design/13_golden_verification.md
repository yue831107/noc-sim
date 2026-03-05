# Golden Data Verification Mechanism

This document describes how golden data is generated and verified for Host Write and Host Read scenarios in the NoC behavior model.

## Overview

The golden data mechanism uses a **capture-at-source** approach:

- **Host Write**: Golden is captured from HostMemory at transfer initiation
- **Host Read**: Golden is captured from LocalMemory at transfer initiation

| Scenario | Golden Source | Capture Timing | Verification Target |
|----------|---------------|----------------|---------------------|
| Host Write | HostMemory | Transfer start | LocalMemory |
| Host Read | LocalMemory | Transfer start | HostMemory |

---

## GoldenManager Class

**File**: `src/core/golden_manager.py`

### GoldenEntry Data Structure

```python
@dataclass
class GoldenEntry:
    node_id: Union[int, str]  # int for node, "host" for HostMemory
    local_addr: int           # Memory address
    data: bytes               # Expected data
    source: GoldenSource      # WRITE_CAPTURE, FILE, or MANUAL
    capture_cycle: int = 0    # Simulation cycle when captured
```

### Core Methods

| Method | Purpose |
|--------|---------|
| `capture_write(node_id, addr, data, cycle)` | Capture golden during write to LocalMemory |
| `capture_gather(host_addr, data_portions, cycle)` | Capture concatenated golden for GATHER |
| `set_golden(node_id, addr, data, source)` | Manually set golden data |
| `get_golden(node_id, addr)` | Get golden data for specific location |
| `get_host_golden(addr)` | Get HostMemory golden (shortcut for `get_golden("host", addr)`) |
| `verify(read_results)` | Compare actual vs expected data |

---

## Host Write Flow

### Sequence

1. `initiate_host_write()` reads payload from `HostMemory[host_addr]`
2. Immediately calls `GoldenManager.capture_golden()` to record expected data
3. Injects the same data into NMU to begin transfer
4. After transfer completes, reads from `LocalMemory[local_addr]` for verification

### Diagram

```
┌─────────────┐  (1) Read    ┌──────────────┐
│ HostMemory  │─────────────►│GoldenManager │  capture_golden()
│ [host_addr] │              │  (Expected)  │
└──────┬──────┘              └──────────────┘
       │                            │
       │ (2) Transfer via Mesh      │ (3) Verify
       ▼                            ▼
┌─────────────┐              ┌──────────────┐
│LocalMemory  │─────────────►│   Compare    │ → PASS/FAIL
│[local_addr] │  (Actual)    │              │
└─────────────┘              └──────────────┘
```

### Code Flow

```python
def submit_write(self, addr, data, axi_id=0):
    # 1. Start transfer via NSU
    result = self.nsu.submit_write(addr, data, axi_id)

    # 2. Capture golden data IMMEDIATELY if submission was successful
    if result:
        node_id = self.address_map.extract_node_id(addr)
        local_addr = self.address_map.extract_local_addr(addr)
        self.golden_manager.capture_write(
            node_id=node_id,
            addr=local_addr,
            data=data,
            cycle=self.current_time
        )

    return result
```

### Verification

```python
def verify_host_write(self, transfer_id):
    # Read actual data from destination (LocalMemory)
    actual_data = self.local_memories[master_id].read(dest_addr, length)

    # Compare with golden
    return self.golden_manager.verify_transfer(transfer_id, actual_data)
```

---

## Host Read Flow

### Sequence

1. `initiate_host_read()` reads expected data from `LocalMemory[local_addr]`
2. Immediately calls `GoldenManager.capture_golden()` to record expected data
3. Sends Read Request to Mesh
4. After transfer completes, reads from `HostMemory[host_addr]` for verification

### Diagram

```
┌─────────────┐  (1) Read    ┌──────────────┐
│LocalMemory  │─────────────►│GoldenManager │  capture_golden()
│[local_addr] │              │  (Expected)  │
└──────┬──────┘              └──────────────┘
       │                            │
       │ (2) Transfer via Mesh      │ (3) Verify
       ▼                            ▼
┌─────────────┐              ┌──────────────┐
│ HostMemory  │─────────────►│   Compare    │ → PASS/FAIL
│ [host_addr] │  (Actual)    │              │
└─────────────┘              └──────────────┘
```

### Code Flow

```python
def initiate_host_read(self, master_id, host_addr, local_addr, length):
    # 1. Read expected data from LocalMemory (source) for golden
    expected_data = self.local_memories[master_id].read(local_addr, length)

    # 2. Capture golden data BEFORE transfer begins
    transfer_id = self.golden_manager.capture_golden(
        source_addr=local_addr,
        dest_addr=host_addr,
        data=expected_data,
        timestamp=self.current_cycle,
        transfer_type="read"
    )

    # 3. Send read request
    self.nmu.start_read_request(config)

    return transfer_id
```

### Verification

```python
def verify_host_read(self, transfer_id):
    # Read actual data from destination (HostMemory)
    actual_data = self.host_memory.read(dest_addr, length)

    # Compare with golden
    return self.golden_manager.verify_transfer(transfer_id, actual_data)
```

---

## Complete Data Flow

### Host Write: HostMemory → LocalMemory

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HOST WRITE FLOW                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HostMemory ──(1) Read──► GoldenManager.capture_golden()                    │
│      │                                                                      │
│      │ (2) Same data                                                        │
│      ▼                                                                      │
│  NMU ──► Mesh (Routers) ──► NSU ──► LocalMemory                   │
│                                                                             │
│  LocalMemory ──(3) Read actual──► GoldenManager.verify_transfer()           │
│                                          │                                  │
│                                          ▼                                  │
│                                     PASS / FAIL                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Host Read: LocalMemory → HostMemory

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HOST READ FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LocalMemory ──(1) Read──► GoldenManager.capture_golden()                   │
│      │                                                                      │
│      │ (2) Transfer data                                                    │
│      ▼                                                                      │
│  NSU ──► Mesh (Routers) ──► NMU ──► HostMemory                    │
│                                                                             │
│  HostMemory ──(3) Read actual──► GoldenManager.verify_transfer()            │
│                                          │                                  │
│                                          ▼                                  │
│                                     PASS / FAIL                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Design Principles

| Principle | Description |
|-----------|-------------|
| **Capture-at-Source** | Golden is always captured from source memory at transfer start |
| **Verify-at-Destination** | Verification reads from destination after transfer completes |
| **Transfer ID Tracking** | Each transfer has unique ID linking golden to metadata |
| **Explicit Verification** | User must call `verify_host_write()` or `verify_host_read()` |
| **Decoupled Timing** | Golden capture and verification are separate operations |

---

## SCATTER/GATHER Mode

For multi-node transfers, GoldenManager uses `(node_id, address)` as key to support different data per node.

### Golden Key Structure

```python
golden_data: Dict[Tuple[Union[int, str], int], bytes]
# Key: (node_id, address) for LocalMemory
# Key: ("host", address) for HostMemory
```

### SCATTER Write

Distributes different data portions to different nodes.

```
Config: mode=SCATTER, target_nodes=[0,1,2,3], size=1024

Capture:
  portion_size = 1024 / 4 = 256 bytes
  golden_data[(0, dst_addr)] = HostMemory[0:256]
  golden_data[(1, dst_addr)] = HostMemory[256:512]
  golden_data[(2, dst_addr)] = HostMemory[512:768]
  golden_data[(3, dst_addr)] = HostMemory[768:1024]

Verify: Each node's LocalMemory[dst_addr] vs its golden portion
```

### GATHER Read

Collects data from multiple nodes into HostMemory.

```
Config: mode=GATHER, target_nodes=[0,1,2,3], size=1024

Capture:
  portion_size = 1024 / 4 = 256 bytes
  golden_data[("host", dst_addr)] = concat(
    LocalMemory[0][src_addr:+256],
    LocalMemory[1][src_addr:+256],
    LocalMemory[2][src_addr:+256],
    LocalMemory[3][src_addr:+256]
  )

Verify: HostMemory[dst_addr] vs concatenated golden
```

### Mode Comparison

| Mode | Golden Source | Golden Key | Verification |
|------|---------------|------------|--------------|
| BROADCAST | Same data to all | `(node_id, addr)` per node | Each LocalMemory |
| SCATTER | Different portions | `(node_id, addr)` per node | Each LocalMemory |
| GATHER | Multiple sources | `("host", addr)` concatenated | HostMemory |

---

## Usage Example

```python
# Setup
system = V1System(mesh_rows=4, mesh_cols=5)
system.host_memory.write(0x0000, payload)

# Host Write with verification
write_id = system.initiate_host_write(
    master_id=0,
    host_addr=0x0000,
    local_addr=0x1000,
    length=len(payload)
)
system.run_until_complete()
assert system.verify_host_write(write_id) == True

# Host Read with verification
read_id = system.initiate_host_read(
    master_id=0,
    host_addr=0x2000,
    local_addr=0x1000,
    length=len(payload)
)
system.run_until_complete()
assert system.verify_host_read(read_id) == True
```

---

## NoC-to-NoC Golden Verification

NoC-to-NoC mode reuses GoldenManager with a new method for generating golden data from node-to-node transfers.

### Comparison: Host-to-NoC vs NoC-to-NoC

| Aspect | Host-to-NoC | NoC-to-NoC |
|--------|-------------|------------|
| Golden Source | HostMemory[src_addr] | Node[src].LocalMemory[src_addr] |
| Golden Target | Node[dst].LocalMemory[dst_addr] | Node[dst].LocalMemory[dst_addr] |
| Key Format | `(dst_node_id, dst_addr)` | `(dst_node_id, dst_addr)` |
| Capture Timing | When HostAXIMaster initiates | Before simulation starts |
| Collision | Possible (BROADCAST to all) | Possible (multiple → same dst) |

### generate_noc_golden() Method

```python
def generate_noc_golden(
    self,
    node_configs: List[NodeTransferConfig],
    get_node_memory: Callable[[int], Memory],
    mesh_cols: int = 5,
) -> int:
    """
    Generate golden data for NoC-to-NoC traffic.

    For each transfer config:
      1. Read source data from Node[src].LocalMemory[src_addr]
      2. Store as golden for Node[dst].LocalMemory[dst_addr]

    Collision handling: Last write wins.
    """
```

### NoC-to-NoC Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NoC-to-NoC Golden Verification Flow                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. configure_traffic(config)                                           │
│     └─→ Generate NodeTransferConfig for each node                       │
│                                                                         │
│  2. initialize_node_memory(pattern="sequential")                        │
│     └─→ Node[i].LocalMemory[src_addr] = unique_data(i)                  │
│                                                                         │
│  3. generate_golden()                                                   │
│     │                                                                   │
│     │  For each NodeTransferConfig:                                     │
│     │    expected = Node[src].LocalMemory[src_addr]                     │
│     │                                                                   │
│     └─→ GoldenManager.store[(dst_node_id, dst_addr)] = expected         │
│                                                                         │
│  4. start_all_transfers() + run_until_complete()                        │
│     └─→ Data flows through mesh                                         │
│                                                                         │
│  5. verify_transfers()                                                  │
│     │                                                                   │
│     │  For each NodeTransferConfig:                                     │
│     │    actual = Node[dst].LocalMemory[dst_addr]                       │
│     │    expected = GoldenManager.get(dst_node_id, dst_addr)            │
│     │                                                                   │
│     └─→ Compare: expected == actual → PASS/FAIL                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### NoCSystem Integration

NoCSystem provides wrapper methods for golden verification:

```python
class NoCSystem:
    def __init__(self, ...):
        self.golden_manager = GoldenManager()

    def generate_golden(self) -> int:
        """Generate golden from source node memories."""
        return self.golden_manager.generate_noc_golden(
            node_configs=self._traffic_config.node_configs,
            get_node_memory=lambda x: self.node_controllers[x].local_memory,
            mesh_cols=self.mesh_cols,
        )

    def verify_transfers(self) -> VerificationReport:
        """Verify actual data against golden."""
        read_results = {}
        for nc in self._traffic_config.node_configs:
            dst_node_id = self._coord_to_node_id(nc.dest_coord)
            actual = self.node_controllers[dst_node_id].read_local_memory(
                nc.local_dst_addr, nc.transfer_size
            )
            read_results[(dst_node_id, nc.local_dst_addr)] = actual
        return self.golden_manager.verify(read_results)
```

### Collision Handling

When multiple sources write to the same destination:

- **Policy**: Last write wins
- Later node_config entry overwrites earlier golden entry

```
Example: Node 0 and Node 2 both send to Node 1 @ 0x1000

Config order:
  [0] Node 0 → Node 1 @ 0x1000  (first write)
  [1] Node 2 → Node 1 @ 0x1000  (second write, overwrites)

Result:
  Golden[(1, 0x1000)] = Node 2's source data
```

---

## Related Files

| File | Purpose |
|------|---------|
| `src/system/golden_manager.h` | GoldenManager class |
| `src/system/noc_system.h` | NocSystem with golden integration |
| `src/memory/memory.h` | Memory classes (HostMemory, LocalMemory) |
| `tests/unit/test_golden_manager.cpp` | GoldenManager unit tests |
| `tests/integration/test_read_verification.cpp` | Read verification integration tests |
