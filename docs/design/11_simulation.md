# 模擬規格

本文件定義模擬的參數設定、I/O pattern 格式、NocSystem public API、DPI-C bridge、平台限制與 cycle model。

---

## 1. 系統配置（NocConfig）

### 1.1 完整 NocConfig 定義

```cpp
struct NocConfig {
    // Topology
    int mesh_cols = 5;
    int mesh_rows = 4;

    // Buffer
    int input_buffer_depth  = 4;       // Router input buffer 深度 (flits)
    int output_buffer_depth = 2;       // Router output buffer 深度 (0 = wire-through)
    int nmu_buffer_depth    = 2;       // NMU injection buffer 深度

    // Flow control (compile-time template, config carries the value)
    FlowControlMode flow_control = FlowControlMode::VALID_READY;
    int num_vc = 1;                    // Virtual channels (Version B)

    // Pipeline delays
    int routing_delay    = 0;          // Extra cycles for route computation
    int vc_alloc_delay   = 0;          // Extra cycles for VC allocation
    int sw_alloc_delay   = 0;          // Extra cycles for switch allocation
    int credit_delay     = 1;          // Credit return latency (cycles)
    int channel_delay    = 1;          // Link propagation latency (cycles)

    // Allocator selection
    std::string vc_allocator = "round_robin";   // "round_robin" | "islip"
    std::string sw_allocator = "qos_aware_rr";  // "qos_aware_rr" | "islip"

    // Traffic (standalone mode)
    int max_outstanding = 8;
    int max_burst_len   = 16;

    // NI
    int rob_entries = 32;              // Reorder Buffer entries per NI

    // Factory helper
    static NocConfig load_from_json(const std::string& json_path);
};
```

### 1.2 參數表

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `mesh_cols` | 5 | Mesh 欄數 |
| `mesh_rows` | 4 | Mesh 列數 |
| `input_buffer_depth` | 4 | Router input buffer 深度 |
| `output_buffer_depth` | 2 | Router output buffer 深度 (0 = wire-through) |
| `nmu_buffer_depth` | 2 | NMU injection buffer 深度 |
| `flow_control` | `VALID_READY` | Flow control mode (compile-time) |
| `num_vc` | 1 | Virtual channel 數量 |
| `routing_delay` | 0 | Route computation 額外 cycle |
| `vc_alloc_delay` | 0 | VC allocation 額外 cycle |
| `sw_alloc_delay` | 0 | Switch allocation 額外 cycle |
| `credit_delay` | 1 | Credit return latency (cycles) |
| `channel_delay` | 1 | Link propagation latency (cycles) |
| `vc_allocator` | `"round_robin"` | VC allocator 策略 |
| `sw_allocator` | `"qos_aware_rr"` | Switch allocator 策略 |
| `max_outstanding` | 8 | 最大未完成交易數 |
| `max_burst_len` | 16 | 最大 burst 長度 |
| `rob_entries` | 32 | RoB entries per NI |

### 1.3 固定設計參數

以下參數為固定值，與 [Flit 格式](02_flit.md) 一致，不可於模擬中調整：

| 參數 | 值 | 說明 |
|------|-----|------|
| `FLIT_WIDTH` | 408 bits | Flit 總寬度 (Header + Payload) |
| `HEADER_WIDTH` | 56 bits | Header 寬度 |
| `PAYLOAD_WIDTH` | 352 bits | 最大 Payload 寬度 |
| `AXI_DATA_WIDTH` | 256 bits (32 bytes) | AXI 資料寬度 |
| `AXI_ADDR_WIDTH` | 64 bits | AXI 位址寬度 |
| `AXI_ID_WIDTH` | 8 bits | AXI transaction ID 寬度 |
| `NODE_ID_WIDTH` | 8 bits | 節點 ID 寬度 ({y[3:0], x[3:0]}) |
| `QOS_WIDTH` | 4 bits | QoS 優先級寬度 |
| `ROB_IDX_WIDTH` | 5 bits | RoB index 寬度 (32 entries) |
| `ECC_WIDTH` | 32 bits | ECC 總寬度 (SECDED) |
| Physical Link | 410 bits | valid + ready + 408-bit flit |

---

## 2. Input / Output Pattern 定義

C++ model 的兩大目標：(1) Pre-silicon performance evaluation（高速模擬 + 參數掃描）；(2) RTL co-simulation golden reference（C++ output 作為 RTL 驗證的比對基準）。

核心流程：**同一組 Input Pattern 分別餵入 C++ model 和 RTL，C++ 的 Output 當 Golden，與 RTL Output 比對。**

### 2.1 I/O 總覽

```
 ┌─────────────────┐                              ┌─────────────────┐
 │  INPUT PATTERNS  │                              │ OUTPUT PATTERNS  │
 │  (使用者提供)     │                              │ (Model 產出)     │
 ├─────────────────┤     ┌──────────────────┐      ├─────────────────┤
 │ 1. Config       │────►│                  │─────►│ 1. Memory State │
 │    (.json)      │     │  C++ NoC Model   │      │    (.hex)       │
 │                 │     │  (golden ref)    │      │                 │
 │ 2. Memory Init  │────►│                  │─────►│ 2. Response Log │
 │    (.hex)       │     │                  │      │    (.json)      │
 │                 │     │                  │      │                 │
 │ 3. Traffic      │────►│                  │─────►│ 3. Cycle Trace  │
 │    (.json)      │     │                  │      │    (.vcd/.json) │
 └─────────────────┘     │                  │      │                 │
                         │                  │─────►│ 4. Statistics   │
                         └──────────────────┘      │    (.json)      │
                                                   └─────────────────┘
 同一組 Input ──────►  RTL Simulation (DUT) ──────► ACTUAL Output
                                                        │
                         ┌──────────────┐               │
              GOLDEN ───►│  Comparator  │◄──── ACTUAL   │
                         │              │               │
                         └──────┬───────┘
                                │
                          PASS / FAIL
                          + diff report
```

### 2.2 I/O Pattern 總覽表

| Pattern | 方向 | 格式 | 用途 | Golden 比對 |
|---------|------|------|------|:----------:|
| Config | INPUT | `.json` | 系統參數 | — |
| Memory Init | INPUT | `.hex` | 初始記憶體 | — |
| Traffic | INPUT | `.json` + `.hex` | 交易序列 + write data | — |
| Memory State | OUTPUT | `.hex` | 最終記憶體 | byte-exact |
| Response Log | OUTPUT | `.json` + `.hex` | 交易回應 + read data | cycle-accurate |
| Cycle Trace | OUTPUT | `.vcd` / `.json` | 時序追蹤 | debug only |
| Statistics | OUTPUT | `.json` | 效能統計 | derived |

### 2.3 Input Pattern 1: Config（系統配置）

**用途**：定義 NoC 拓撲、buffer 深度、flow control 等硬體參數。
**格式**：JSON
**消費者**：C++ model constructor / RTL testbench parameter override

```json
{
  "topology": {
    "mesh_cols": 5,
    "mesh_rows": 4
  },
  "router": {
    "input_buffer_depth": 4,
    "output_buffer_depth": 2,
    "num_vc": 1,
    "flow_control": "valid_ready",
    "vc_allocator": "round_robin",
    "sw_allocator": "qos_aware_rr"
  },
  "ni": {
    "nmu_buffer_depth": 2,
    "rob_entries": 32,
    "max_burst_len": 16
  },
  "pipeline": {
    "routing_delay": 0,
    "vc_alloc_delay": 0,
    "sw_alloc_delay": 0,
    "credit_delay": 1,
    "channel_delay": 1
  }
}
```

**RTL 映射**：JSON → SV testbench 透過 DPI-C 讀取，或轉為 Verilog `parameter` / `define`。

### 2.4 Input Pattern 2: Memory Init（記憶體初始狀態）

**用途**：預載 HostMemory 和各 node 的 LocalMemory 初始內容。
**格式**：Verilog `$readmemh` 格式（`.hex`）
**消費者**：C++ `load_memory()` / RTL `$readmemh()`

```
// host_memory.hex — HostMemory 初始內容
// 格式: @<hex_addr> <hex_data_bytes>
@00000000 AB CD EF 01 02 03 04 05 ...
@00000100 11 22 33 44 55 66 77 88 ...

// local_memory_node_0x10.hex — Node (0,1) 的 LocalMemory
@00001000 FF FF FF FF 00 00 00 00 ...
```

**說明**：
- 每個 node 一個 `.hex` 檔（可選，未指定則為 0）
- HostMemory 一個 `.hex` 檔
- C++ model 和 RTL testbench 讀取相同的 `.hex` 檔

### 2.5 Input Pattern 3: Traffic（交易序列）

**用途**：定義要執行的 AXI transaction 序列。
**格式**：JSON
**消費者**：C++ TrafficManager / RTL testbench AXI driver

```json
{
  "transactions": [
    {
      "id": 0,
      "type": "WRITE",
      "src_node": "0x00",
      "dst_addr": "0x10_00001000",
      "data_file": "txn0_wdata.hex",
      "data_size": 64,
      "axi_id": 0,
      "burst_len": 2,
      "inject_cycle": 0
    },
    {
      "id": 1,
      "type": "READ",
      "src_node": "0x00",
      "dst_addr": "0x10_00001000",
      "data_size": 64,
      "axi_id": 1,
      "burst_len": 2,
      "inject_cycle": "after:0"
    },
    {
      "id": 2,
      "type": "MULTICAST_WRITE",
      "src_node": "0x00",
      "dst_nodes": ["0x21", "0x22", "0x31", "0x32"],
      "dst_addr": "0x00002000",
      "data_file": "txn2_wdata.hex",
      "data_size": 128,
      "axi_id": 2,
      "inject_cycle": 100
    }
  ]
}
```

**欄位說明**：

| 欄位 | 說明 |
|------|------|
| `id` | Transaction 唯一識別碼 |
| `type` | `WRITE` / `READ` / `MULTICAST_WRITE` |
| `src_node` | 發起 transaction 的 NMU 所在 node ID |
| `dst_addr` | 64-bit AXI 地址（`[39:32]`=node_id, `[31:0]`=local_addr） |
| `dst_nodes` | Multicast 目標 node list（僅 MULTICAST_WRITE） |
| `data_file` | Write data 的 `.hex` 檔路徑 |
| `data_size` | 傳輸大小 (bytes) |
| `axi_id` | AXI transaction ID |
| `burst_len` | AXI burst length (beats) |
| `inject_cycle` | 注入時機：絕對 cycle 數 或 `"after:<txn_id>"` 表示等前一筆完成 |

### 2.6 Output Pattern 1: Memory State（最終記憶體狀態）

**用途**：所有 transaction 完成後，各 node LocalMemory 的完整 dump。作為 golden 比對基準。
**格式**：`.hex`（與 Input Memory 相同格式）
**比對方式**：byte-level exact match

```
// golden_local_memory_node_0x10.hex
@00001000 AB CD EF 01 02 03 04 05 ...
```

**C++ 產生**：`system.dump_memory_state(output_dir)` → 每個 node 產出一個 `.hex` 檔
**RTL 產生**：testbench `$writememh()` 或 DPI-C dump

### 2.7 Output Pattern 2: Response Log（交易回應記錄）

**用途**：每筆 transaction 的 AXI response 記錄。用於比對 response 正確性與順序。
**格式**：JSON
**比對方式**：per-transaction status + data match

```json
{
  "responses": [
    {
      "txn_id": 0,
      "type": "WRITE",
      "status": "OKAY",
      "complete_cycle": 12,
      "b_resp": "OKAY"
    },
    {
      "txn_id": 1,
      "type": "READ",
      "status": "OKAY",
      "complete_cycle": 25,
      "r_data_file": "golden_txn1_rdata.hex",
      "r_resp": "OKAY"
    },
    {
      "txn_id": 2,
      "type": "MULTICAST_WRITE",
      "status": "OKAY",
      "complete_cycle": 156,
      "b_resp": "MC_ALL_OK"
    }
  ]
}
```

**比對欄位**：

| 欄位 | 比對方式 |
|------|---------|
| `status` | Exact match（OKAY / SLVERR / DECERR） |
| `b_resp` | Exact match |
| `r_data_file` | Byte-level exact match with golden |
| `complete_cycle` | **Cycle-accurate match**（C++ model 與 RTL 同 cycle 完成） |

### 2.8 Output Pattern 3: Cycle Trace（時序追蹤，可選）

**用途**：Debug 用途。記錄每 cycle 每個 router/NI/channel 的狀態。
**格式**：VCD（Value Change Dump）或自定義 JSON trace
**比對方式**：通常不做自動比對，用於人工 debug

```json
{
  "cycle": 5,
  "routers": {
    "(0,0)": {
      "input_buffer": {"EAST": [{"flit_id": 3, "vc": 0}], "LOCAL": []},
      "output_valid": {"EAST": true, "NORTH": false},
      "path_locks": {"LOCAL": {"state": "LOCKED", "output": "EAST"}}
    }
  },
  "channels": {
    "(0,0)-E-(1,0)": {"req": {"valid": true, "flit_id": 2}, "rsp": {"valid": false}}
  }
}
```

**VCD 替代方案**：每個 router port 的 valid/ready/flit 信號轉為 VCD 波形，可用 GTKWave 檢視。

### 2.9 Output Pattern 4: Statistics（效能統計）

**用途**：效能評估，不用於 golden 比對（C++ 與 RTL 的 cycle count 必須相同，但 statistics 是衍生值）。
**格式**：JSON

```json
{
  "summary": {
    "total_cycles": 200,
    "total_transactions": 3,
    "avg_latency": 64.3,
    "throughput_gbps": 12.5
  },
  "per_transaction": [
    {"txn_id": 0, "latency": 12, "hops": 3},
    {"txn_id": 1, "latency": 25, "hops": 6},
    {"txn_id": 2, "latency": 156, "hops": "N/A (multicast)"}
  ],
  "per_router": {
    "(0,0)": {"flits_forwarded": 15, "buffer_peak_occupancy": 3},
    "(1,0)": {"flits_forwarded": 12, "buffer_peak_occupancy": 2}
  }
}
```

---

## 3. 4 層架構

本系統分為 4 層：

```
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 1: User Code / RTL Testbench                                  │
│   使用者撰寫的 test 程式或 SV testbench                              │
└────────────────────────────┬────────────────────────────────────────┘
                             ↓ calls
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 2: NocSystem Public API (noc_api.h)                           │
│   Transaction / Simulation Control / Query / Debug                  │
└────────────────────────────┬────────────────────────────────────────┘
                             ↓ delegates to
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 3: Internal Components                                        │
│   TrafficManager / Mesh / Router / NI / Channel / Memory / Stats    │
│                                                                     │
│   ┌─────────────────────── HOT-SWAP BOUNDARY ──────────────────┐   │
│   │  Router_Interface<Mode>  /  NI_Interface<Mode>              │   │
│   │  CppRouter ↔ RtlRouterProxy  |  CppNI ↔ RtlNIProxy       │   │
│   └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────┘
                             ↓ DPI-C
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 4: Co-Sim Bridge + RTL                                        │
│   DPI-C functions / RtlProxy / SystemVerilog modules                │
└─────────────────────────────────────────────────────────────────────┘
```

詳細架構圖見 [docs/images/noc_env.txt](../images/noc_env.txt)。

---

## 4. NocSystem Public API（Layer 2）

NocSystem 的 public API 分為 6 組。

### Group A: Construction & Configuration

```cpp
class NocSystem {
public:
    explicit NocSystem(const NocConfig& config);
    ~NocSystem();

    // === Input Pattern Loading ===
    static NocConfig load_config(const std::string& json_path);     // Config
    void load_memory(const std::string& hex_dir);                   // Memory Init
    void load_traffic(const std::string& json_path);                // Traffic

    // Direct memory access
    void load_host_memory(uint64_t addr, const uint8_t* data, size_t len);
    void load_local_memory(uint8_t node_id, uint32_t addr,
                           const uint8_t* data, size_t len);
    void dump_local_memory(uint8_t node_id, uint32_t addr,
                           uint8_t* buf, size_t len) const;
```

### Group B: Transaction Submission（High-Level）

```cpp
    // High-level: 自動拆 burst、管理 AXI channel sequencing
    // 回傳 transaction handle（用於追蹤 completion）
    TxnHandle submit_write(uint64_t addr, const uint8_t* data, size_t len,
                           uint8_t axi_id = 0);
    TxnHandle submit_read(uint64_t addr, size_t len, uint8_t axi_id = 0);
    TxnHandle submit_multicast_write(uint64_t base_addr, const uint8_t* data,
                                     size_t len,
                                     const std::vector<uint8_t>& dst_nodes);
```

### Group C: Simulation Control

```cpp
    void process_cycle();                              // 1 cycle (8 phases)
    void run(int cycles);                              // N cycles
    int  run_until_idle(int max_cycles = 1000000);     // 直到 network idle
    void run_all();                                    // 執行所有 loaded traffic
    int  current_cycle() const;
```

### Group D: Completion & Response

```cpp
    // Polling-based
    bool is_complete(TxnHandle h) const;
    bool all_complete() const;
    TxnStatus get_status(TxnHandle h) const;    // PENDING / COMPLETE / ERROR
    std::vector<uint8_t> get_read_data(TxnHandle h) const;

    // Callback-based
    using CompletionCallback = std::function<void(TxnHandle, TxnStatus)>;
    void set_completion_callback(CompletionCallback cb);
```

### Group E: Metrics & Verification

```cpp
    const MetricsCollector& get_metrics() const;
    VerificationReport verify() const;
```

### Group F: Debug & Introspection

```cpp
    // 存取內部元件（read-only introspection）
    const Router_Interface<Mode>* get_router(Coord pos) const;
    const NI_Interface<Mode>*     get_ni(Coord pos) const;
    const Mesh<Mode>&             get_mesh() const;

    // State dump（for waveform / debug）
    void dump_state(std::ostream& os) const;

    // === Output Pattern Generation ===
    void dump_memory_state(const std::string& hex_dir) const;       // Memory State
    void dump_response_log(const std::string& json_path) const;     // Response Log
    void dump_cycle_trace(const std::string& vcd_path) const;       // Cycle Trace
    void dump_statistics(const std::string& json_path) const;       // Statistics

    // === Convenience: 一鍵 golden generation ===
    void generate_golden(const std::string& output_dir) const;      // 產生所有 output patterns
};
```

### 4.1 TxnHandle 與 TxnStatus

```cpp
using TxnHandle = uint32_t;

enum class TxnStatus {
    PENDING,           // 已提交，尚未完成
    INJECTING,         // 正在注入 flits
    WAITING_RESPONSE,  // 所有 flits 已注入，等待 response
    COMPLETE,          // Transaction 完成（OKAY）
    ERROR              // Transaction 完成（SLVERR / DECERR）
};
```

### 4.2 使用範例

```cpp
// 1. 從檔案載入
auto config = NocSystem::load_config("patterns/config.json");
NocSystem system(config);
system.load_memory("patterns/");
system.load_traffic("patterns/traffic.json");

// 2. 執行所有 traffic
system.run_all();

// 3. 產生 golden output
system.generate_golden("golden/");

// 4. 驗證
auto report = system.verify();
assert(report.passed);
```

```cpp
// 或手動 API 模式
NocConfig config;
config.mesh_cols = 5;
config.mesh_rows = 4;
NocSystem system(config);

// 提交 write transaction (64B write to node (1,0) addr 0x1000)
std::vector<uint8_t> data(64, 0xAB);
uint64_t addr = (0x10ULL << 32) | 0x1000;
auto h = system.submit_write(addr, data.data(), data.size(), /*axi_id=*/0);

// 執行直到完成
system.run_until_idle();
assert(system.get_status(h) == TxnStatus::COMPLETE);

// 查看效能
auto& metrics = system.get_metrics();
```

### 4.3 RTL Testbench 對應流程

```
SystemVerilog Testbench:
  1. 讀取 Config JSON（via DPI-C）→ 設定 RTL parameters
  2. $readmemh() 載入 Memory Init → 預載 RTL memory
  3. 讀取 Traffic JSON（via DPI-C）→ 驅動 AXI transactions
  4. 執行 simulation
  5. $writememh() dump RTL Memory State
  6. 收集 AXI responses → 寫入 Response Log
  7. 比對: diff golden/ vs actual/
```

---

## 5. DPI-C Bridge Interface（Layer 4）

DPI-C bridge 為 Layer 4 的 thin wrapper，提供 SystemVerilog 與 C++ model 的互操作介面。

### 5.1 Function Signatures

```cpp
extern "C" {
    // Construction
    void* noc_init(const char* config_json);                          // JSON path → opaque handle
    void  noc_destroy(void* handle);

    // Transaction submission
    int   noc_submit_write(void* h, uint64_t addr,
                           const uint8_t* data, uint32_t len);        // → TxnHandle
    int   noc_submit_read(void* h, uint64_t addr, uint32_t len);     // → TxnHandle

    // Simulation
    void  noc_step(void* h, int cycles);

    // Response
    int   noc_get_response(void* h, uint8_t* buf, uint32_t buf_size); // 0=none, 1=write, 2=read
    int   noc_pending_count(void* h);
}
```

### 5.2 RTL Co-simulation 流程

```
SystemVerilog Testbench          DPI-C Bridge          C++ NocSystem
═══════════════════════         ════════════          ═══════════════
  initial begin                      │                      │
    handle = noc_init(json); ──────► │ ────────────► load_config + NocSystem(cfg)
    noc_submit_write(...);   ──────► │ ────────────► submit_write(...)
    repeat(N) begin                  │                      │
      @(posedge clk);               │                      │
      noc_step(handle, 1);  ──────► │ ────────────► process_cycle()
      status = noc_get_response(); ►│ ────────────► poll response
    end                              │                      │
    noc_destroy(handle);     ──────► │ ────────────► delete system
  end                                │                      │
```

### 5.3 Memory Model

DPI-C bridge 下，C++ model 的 local memory 與 RTL 的 memory 需保持同步：

| 模式 | Memory 歸屬 | 同步機制 |
|------|-----------|---------|
| **C++ as golden** | C++ model 維護 memory | RTL memory 由 C++ model shadow |
| **RTL as primary** | RTL memory 為 primary | C++ model 每 cycle 讀取 RTL memory state |
| **Independent** | 各自維護 | 比對 response data 驗證一致性 |

---

## 6. 模擬平台限制

本 C++ model 為 **cycle-accurate behavioral model**，模擬 NoC 的功能行為與 timing，但不模擬硬體物理特性。

### 6.1 模擬範圍

| 模擬項目 | 是否模擬 | 說明 |
|---------|:-------:|------|
| Cycle-accurate timing | ✓ | 8-phase pipeline per cycle |
| Wormhole switching | ✓ | Path lock/release FSM |
| Credit-based flow control | ✓ | Per-VC credit counters (Version B) |
| Valid/Ready handshake | ✓ | AXI-style handshake (Version A) |
| AXI protocol | ✓ | AW/W/AR/B/R channels, burst, reorder |
| QoS-Aware arbitration | ✓ | QoS priority + Round-Robin |
| ECC generate/check | ✓ | SECDED 行為模型 |
| Multicast/Broadcast | ✓ | In-network reduction（無 timeout） |
| Backpressure propagation | ✓ | Per-hop ready/credit backpressure |
| Configurable channel delay | ✓ | Channel\<T\> with latency parameter |
| Clock domain crossing | ✗ | 假設全域同步時脈 |
| Power/voltage | ✗ | 不模擬功耗 |
| Physical wire delay | ✗ | Channel delay 模擬 hop latency |
| FIFO physical implementation | ✗ | 行為模型（std::deque） |
| Deadlock recovery | ✗ | 依賴 XY routing 的 deadlock freedom |
| Process variation | ✗ | 不模擬製程變異 |
| Clock gating | ✗ | 不模擬 |
| Reset sequence | ✗ | 假設初始狀態已就緒 |

### 6.2 Timing 抽象

| 硬體行為 | 模擬抽象 |
|---------|---------|
| Combinational logic delay | 同 phase 內瞬間完成 |
| Wire propagation delay | Channel latency 參數（預設 1 cycle） |
| FF setup/hold time | 不模擬 |
| Metastability | 不模擬（同步假設） |
| Pipeline register | Channel delay + output buffer |

### 6.3 已知限制

1. **Single-cycle router pipeline**：RTL 中 RC→VA→SA→ST 可能需多 cycle pipeline；本模型在單一 Phase 4 內完成，可透過 `routing_delay` / `vc_alloc_delay` / `sw_alloc_delay` 加入額外 cycle
2. **Channel delay**：Channel\<T\> 的 `latency` 參數模擬 link propagation delay（預設 1 cycle），取代原本的 zero-cycle wire_all()
3. **Ideal memory**：Local memory read/write 在單一 cycle 內完成（無 memory access latency）
4. **No contention on AXI bus**：AXI slave 總是就緒（無 AXI slave stall 模擬）

---

## 7. Replaceable Components（可替換邊界）

### 7.1 Hot-Swap Boundary（Abstract Interface）

```
                    ┌─────── HOT-SWAP BOUNDARY ───────┐
                    │                                   │
CppRouter ──────────┤ Router_Interface<Mode>            │──── RtlRouterProxy
                    │   get_output / set_input / tick   │
                    │                                   │
CppNI ──────────────┤ NI_Interface<Mode>                │──── RtlNIProxy
                    │   get_output / set_input / tick   │
                    │                                   │
                    └───────────────────────────────────┘
```

### 7.2 Pluggable Components（Factory Pattern）

| 元件 | Abstract Base | 可選實作 | 切換方式 |
|------|-------------|---------|---------|
| Router | `Router_Interface<Mode>` | CppRouter, RtlRouterProxy | **Hot-swap**（per-instance） |
| NI | `NI_Interface<Mode>` | CppNI, RtlNIProxy | **Hot-swap**（per-instance） |
| VC Allocator | `Allocator` | RoundRobin, iSLIP | Config string |
| SW Allocator | `Allocator` | QoSAwareRR, iSLIP | Config string |
| Traffic Pattern | `TrafficPattern` | Uniform, Tornado, Custom | Config string |
| Buffer Policy | `BufferPolicy` | Private, Shared | Config string |

### 7.3 不可替換（固定 C++ 實作）

| 元件 | 原因 |
|------|------|
| TrafficManager | 中央協調，不需替換 |
| Mesh topology | 固定 mesh，由 config 參數化 |
| Channel | Link model，不需替換 |
| GoldenManager | 驗證邏輯，C++ only |
| MetricsCollector | 統計收集，C++ only |
| HostMemory / LocalMemory | 行為模型，不需替換 |

---

## 8. 8-Phase Cycle Model 精確定義

本 section 集中定義 cycle model，為所有文件的 canonical reference。詳細 phase 行為見 [Internal Interface](05_internal_interface.md) Section 7。

### 8.1 Phase 總覽

| Phase | 名稱 | 動作 | 涉及元件 |
|-------|------|------|---------|
| 1 | Sample | 各 input port latch incoming flit（`in_valid && out_ready` → push to input buffer） | Router input ports |
| 2 | Clear Inputs | 清除上 cycle 的 `in_valid`/`in_flit` 信號（防止重複 sample） | Router input ports |
| 3 | Update Ready | `out_ready = !input_buffer.full()`（combinational） | Router input ports |
| 4 | Route & Forward | XY routing + QoS-Aware arbitration + crossbar switching + wormhole lock/unlock | Router core logic |
| 5 | Wire All | 透過 Channel\<T\> 交換所有元件的 output → 對端 input | Simulation Driver |
| 6 | Clear Accepted | 若 `out_valid && in_ready`：清除 output（handshake 完成） | Router output ports |
| 7 | Credit Release | 歸還 credit 給上游（Version B）；Version A 為 no-op | Credit counters |
| 8 | NI Process | NMU/NSU 處理 AXI ↔ Flit 轉換、injection、reassembly | NI (NMU + NSU) |

### 8.2 Phase 因果關係

```
Phase 1 (Sample)
  ↓ buffer 狀態更新
Phase 2 (Clear) → Phase 3 (Ready)
                     ↓ ready 信號 + buffer 內容
                  Phase 4 (Route & Forward)
                     ↓ out_valid/out_flit 設定
                  Phase 5 (Propagate via Channel) ──► 延遲後可見
                     ↓ in_ready 可見
                  Phase 6 (Clear Accepted)
                     ↓ buffer slot 釋放
                  Phase 7 (Credit Release) ────► 上游 credit +1
                     ↓
                  Phase 8 (NI Process)
                     ↓ NI output 設定
                  ──────────► 下一 cycle Phase 5 propagate NI signals
```

### 8.3 RTL 映射

| C++ Phase | RTL 對應 | 邏輯類型 |
|-----------|---------|---------|
| Phase 1 (Sample) | `posedge clk`: input FF latch | Sequential |
| Phase 2 (Clear) | Model housekeeping（無直接 RTL 對應） | — |
| Phase 3 (Ready) | `assign out_ready = ~buffer_full` | Combinational |
| Phase 4 (Route) | RC → VA → SA → ST pipeline | Combinational* |
| Phase 5 (Propagate) | Wire connections via Channel delay | Combinational/Registered |
| Phase 6 (Clear Accepted) | `posedge clk`: output FF update | Sequential |
| Phase 7 (Credit) | `posedge clk`: credit counter update | Sequential |
| Phase 8 (NI) | NI pipeline stages | Mixed |

> *Phase 4 在 C++ model 中為 single-phase behavioral model。RTL 實作可能將 RC/VA/SA/ST 拆為多級 pipeline。可透過 NocConfig 的 pipeline delay 參數模擬多 cycle pipeline。

---

## 相關文件

- [系統概述](01_overview.md)
- [Router 規格](03_router.md) — 8-phase pipeline 行為、CppRouter 內部 pipeline stages
- [Network Interface 規格](04_network_interface.md) — NMU/NSU 處理、CppNI 內部 functions
- [Flit 格式](02_flit.md) — 固定參數基準
- [內部介面架構](05_internal_interface.md) — Channel\<T\>、TrafficManager、Allocator、Phase 詳細定義
- [效能指標](14_metrics.md) — MetricsCollector
- [Golden 驗證](13_golden_verification.md) — VerificationReport
- [架構圖](../images/noc_env.txt) — 系統架構圖
