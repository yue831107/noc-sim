# NoC C++ Model - Project Goals

## 1. 專案背景

本專案目標是建立一套 NoC (Network-on-Chip) 的 C++ Behavior Model，架構規格如下：

- **Topology**: 5×4 Mesh（Uniform Router，每個 Router 可擁有 0~4 個 LOCAL port）
- **Routing**: XY Routing + Wormhole Switching（deadlock-free）
- **Flow Control**: 雙模式 — Valid/Ready (Version A) 和 Credit-Based (Version B)，compile-time 選擇
- **Interface**: AXI4 (AW/W/AR/B/R 五通道)
- **Physical Channel**: Req/Rsp 物理分離（雙軌架構）
- **Flit**: 固定 408-bit（Header 56b + Payload 352b）
- **Modes**: Host-to-NoC（Host 對節點傳輸）

此 C++ model 服務於以下兩大核心目標。

---

## 2. 核心目標

### 目標 A: 硬體前效能評估 (Pre-Silicon Performance Evaluation)

**動機**: 在 RTL 實作之前，以高速 C++ model 進行大規模 traffic pattern 掃描與參數空間探索，提供設計決策依據。

| 需求 | 說明 |
|------|------|
| **高速模擬** | C++ cycle-accurate model |
| **參數掃描** | 支援 mesh size、buffer depth、routing algorithm、pipeline delay 等參數化配置 |
| **Traffic Pattern 評估** | neighbor, shuffle, bit_reverse, random, transpose 等模式 |
| **統計收集** | Throughput, Latency, Buffer Utilization, Flit Conservation |
| **批量測試** | 大規模自動化效能測試，輸出 JSON 報告 |
| **可視化介面** | 輸出資料供外部工具繪圖 |

**預期產出**:
- 不同拓撲/參數組合的效能比較報告
- 瓶頸分析（哪些 router/link 成為 hotspot）
- QoS 策略評估數據
- 為 RTL 設計提供最佳參數建議

---

### 目標 B: RTL 共模驗證 (RTL Co-Simulation / Golden Reference)

**動機**: C++ model 作為 RTL 的 golden reference，確保硬體實作與行為模型一致。

| 需求 | 說明 |
|------|------|
| **Cycle-Accurate** | 與 RTL 逐 cycle 比對，結果必須完全一致 |
| **Simulator 介面** | 支援 DPI-C (SystemVerilog) 與 VPI (Verilog) 雙介面 |
| **Transaction 驅動** | 從 C++ 端產生 AXI transaction，驅動 RTL testbench |
| **Response 驗證** | 攔截 RTL response，與 C++ model 預期結果比對 |
| **Data Integrity** | Golden Data 機制，支援 MULTICAST |
| **Waveform Dump** | 可選擇輸出 C++ model 內部狀態供 waveform 分析 |

**核心流程**: 同一組 Input Pattern 分別餵入 C++ model 和 RTL，C++ 的 Output 當 Golden，與 RTL Output 比對。

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
                         └──────┬───────┘               │
                                │                       │
                          PASS / FAIL                   │
                          + diff report                 │
```

---

## 3. 設計原則

### 3.1 嚴格分層 (4-Layer Architecture)

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

**關鍵**: Layer 3 的 Internal Components 完全不依賴 simulator，可獨立編譯為 library。

### 3.2 Abstract Interface + Hot-Swap

透過 `Router_Interface<Mode>` 和 `NI_Interface<Mode>` 抽象介面，支援：
- **CppRouter / CppNI**: 純 C++ 行為模型（用於獨立效能模擬）
- **RtlRouterProxy / RtlNIProxy**: 透過 DPI-C 橋接到 RTL（用於 co-simulation）
- 可 per-instance 替換：例如 19 個 CppRouter + 1 個 RtlRouterProxy

Flow control mode 為 compile-time template parameter，不產生未使用的信號：
- `PortOutputVR`：valid + ready，無 credit
- `PortOutputCredit`：credit return，無 ready

### 3.3 Channel-Based Interconnect

Router 之間、NI 與 Router 之間使用 `Channel<T>` 連接，取代 zero-cycle 直連：
- **latency=1**（預設）：下一 cycle 可見，等同 pipeline register
- **latency=0**：combinational wire
- **latency>1**：模擬 long-wire 或額外 pipeline stage

### 3.4 Pluggable Components (Factory Pattern)

以下元件可透過 config string 在 runtime 選擇實作：

| 元件 | Abstract Base | 可選實作 |
|------|-------------|---------|
| VC Allocator | `Allocator` | RoundRobin, iSLIP |
| SW Allocator | `Allocator` | QoSAwareRR, iSLIP |
| Traffic Pattern | `TrafficPattern` | Uniform, Tornado, Custom |
| Buffer Policy | `BufferPolicy` | Private, Shared |

### 3.5 Callback + Polling 雙模式

```cpp
// Polling-based
bool is_complete(TxnHandle h) const;
TxnStatus get_status(TxnHandle h) const;
std::vector<uint8_t> get_read_data(TxnHandle h) const;

// Callback-based
using CompletionCallback = std::function<void(TxnHandle, TxnStatus)>;
void set_completion_callback(CompletionCallback cb);
```

### 3.6 最小化 RTL

- **C++ 負責**: Routing 決策、Flow Control、Flit 組裝/解析、Arbitration、Buffer 管理
- **RTL 負責**: 物理通道傳輸、時序對齊、信號驅動
- DPI-C bridge 保持薄層（僅做 `noc_init` / `noc_step` / `noc_submit_*` / `noc_get_*` 轉發）

---

## 4. 系統架構

### 4.1 I/O Pattern 定義

| Pattern | 方向 | 格式 | 用途 | Golden 比對 |
|---------|------|------|------|:----------:|
| Config | INPUT | `.json` | 系統參數（mesh size, buffer depth, pipeline delay 等） | — |
| Memory Init | INPUT | `.hex` | 初始記憶體（HostMemory + 各 node LocalMemory） | — |
| Traffic | INPUT | `.json` + `.hex` | 交易序列 + write data | — |
| Memory State | OUTPUT | `.hex` | 最終記憶體 dump | ✓ byte-exact |
| Response Log | OUTPUT | `.json` + `.hex` | 交易回應 + read data | ✓ cycle-accurate |
| Cycle Trace | OUTPUT | `.vcd` / `.json` | 時序追蹤（debug only） | ✗ |
| Statistics | OUTPUT | `.json` | 效能統計（derived） | ✗ |

範例 pattern 檔案見 `examples/patterns/`。

### 4.2 NocSystem Public API（6 組）

| Group | 名稱 | 主要 API |
|-------|------|---------|
| **A** | Construction & Configuration | `NocSystem(config)`, `load_host_memory()`, `load_local_memory()`, `dump_local_memory()` |
| **B** | Transaction Submission | `submit_write()`, `submit_read()`, `submit_multicast_write()` → 回傳 `TxnHandle` |
| **C** | Simulation Control | `process_cycle()`, `run(N)`, `run_until_idle()`, `current_cycle()` |
| **D** | Completion & Response | `is_complete()`, `all_complete()`, `get_status()`, `get_read_data()`, `set_completion_callback()` |
| **E** | Metrics & Verification | `get_metrics()`, `verify()` |
| **F** | Debug & Introspection | `get_router(coord)`, `get_ni(coord)`, `get_mesh()`, `dump_state()` |

**Convenience API**:
```cpp
// 一鍵載入 + 執行 + 產出
static NocConfig load_config(const std::string& json_path);
void load_memory(const std::string& hex_dir);
void load_traffic(const std::string& json_path);
void run_all();
void generate_golden(const std::string& output_dir) const;
```

### 4.3 Internal Components

| 元件 | 職責 |
|------|------|
| `NocSystem` | 頂層系統，協調所有元件，暴露 6 組 API |
| `TrafficManager` | 中央協調器：Transaction → Packet → Flit 拆解、injection queue、completion tracking |
| `Mesh` | 拓撲組裝 + 連線管理 + 8-Phase cycle 驅動 |
| `CppRouter` | 6-stage pipeline: InputQueuing → RouteEvaluate → VCAllocEvaluate → SWAllocEvaluate → SwitchTraverse → OutputQueuing |
| `CppNI` | NMU（Pack AW/W/AR, Unpack B/R, Inject）+ NSU（Unpack AW/W/AR, Pack B/R, Reassembly, MemOp）+ ECC/QoS |
| `Channel<T>` | 帶可配置延遲的 link model，取代 wire_all() 直連 |
| `Allocator` | 抽象 arbiter base → RoundRobin / iSLIP / QoSAwareRR |
| `InputBuffer` | Per-port, per-VC data storage |
| `BufferState` | Credit tracking（與 Buffer 分離） |
| `Flit` | 408-bit flit 結構 + object pooling |
| `RouteCompute` | XY routing + multicast RCR |
| `GoldenManager` | Write capture + read verification |
| `MetricsCollector` | Per-node, per-class 效能統計 |
| `HostMemory` / `LocalMemory` | 記憶體模型 |

### 4.4 Replaceable Boundary 總覽

**Hot-Swap（per-instance 替換）**:

| 元件 | Abstract Interface | C++ 實作 | RTL Proxy |
|------|-------------------|---------|-----------|
| Router | `Router_Interface<Mode>` | `CppRouter` | `RtlRouterProxy` |
| NI | `NI_Interface<Mode>` | `CppNI` | `RtlNIProxy` |

**Pluggable（Config string 選擇）**: VC Allocator, SW Allocator, Traffic Pattern, Buffer Policy

**固定 C++ 實作（不可替換）**: TrafficManager, Mesh, Channel, GoldenManager, MetricsCollector, HostMemory/LocalMemory

### 4.5 DPI-C Bridge API

```cpp
extern "C" {
    void* noc_init(const char* config_json);
    int   noc_submit_write(void* h, uint64_t addr, const uint8_t* data, uint32_t len);
    int   noc_submit_read(void* h, uint64_t addr, uint32_t len);
    void  noc_step(void* h, int cycles);
    int   noc_get_response(void* h, uint8_t* buf, uint32_t buf_size);
    int   noc_pending_count(void* h);
    void  noc_destroy(void* h);
}
```

### 4.6 8-Phase Cycle Model

每個 simulation cycle 依序執行以下 8 個 phase：

| Phase | 名稱 | 動作 |
|-------|------|------|
| 1 | **Sample** | 各 router/NI 鎖存輸入 |
| 2 | **Clear** | 清除前一 cycle 的一次性信號 |
| 3 | **Ready/Grant** | 計算 ready 或 grant 信號 |
| 4 | **Route & Forward** | XY 路由 + Wormhole arbitration + crossbar traversal |
| 5 | **Wire** | Channel 傳播（delay line shift） |
| 6 | **Clear Accepted** | 清除已被接收的 flit |
| 7 | **Credit Release** | 釋放 credit / deassert backpressure |
| 8 | **NI Process** | NMU injection + NSU ejection + memory ops |

---

## 5. 目錄結構

```
noc_c_model/
├── docs/                       # 設計文件
│   ├── design/                 # 架構規格（18 份）
│   └── images/                 # 圖片資源（含 noc_env.txt 架構圖）
├── include/                    # Public headers
│   └── noc/
│       ├── noc_api.h           # 使用者 API（6 組）
│       ├── types.h             # 共用型別定義
│       └── config.h            # NocConfig
├── src/                        # C++ 源碼
│   ├── core/                   # Router, NI, Flit, Buffer, Arbiter, Routing, Mesh
│   ├── memory/                 # LocalMemory, HostMemory
│   ├── system/                 # NocSystem, TrafficManager, GoldenManager
│   ├── stats/                  # MetricsCollector
│   └── bridge/                 # DPI-C and VPI simulator bridge
├── rtl/                        # RTL Harness (最小化 SystemVerilog)
├── tests/                      # 測試
│   ├── unit/                   # 單元測試 (GoogleTest)
│   ├── integration/            # 整合測試
│   └── cosim/                  # Co-simulation 測試
├── examples/                   # 使用範例
│   ├── patterns/               # I/O pattern 範例（config.json, traffic.json, .hex）
│   ├── standalone/             # 純 C++ 效能測試
│   └── cosim/                  # RTL Co-simulation 範例
├── lib/                        # 編譯產出
└── CMakeLists.txt              # Build System
```

**Namespace**: `noc::`

---

## 6. 分階段實作計畫

### Phase 1: Core Library（獨立 C++ model）

**目標**: 建立 NoC 核心邏輯，可獨立執行效能模擬。

- [ ] 基礎資料結構: `Flit`（408-bit）、`AXITransaction`、`NocConfig`
- [ ] `Channel<T>` — 帶可配置延遲的 link model
- [ ] `InputBuffer` + `BufferState`（data/credit 分離）
- [ ] `Allocator` 抽象 + RoundRobin / iSLIP 實作
- [ ] `CppRouter`（6-stage pipeline: Route → VCAlloc → SWAlloc → Switch）
- [ ] `CppNI`（NMU pack/inject + NSU unpack/reassembly/memop）
- [ ] `Mesh` topology 組裝 + 8-Phase cycle model
- [ ] `TrafficManager`（Transaction lifecycle 管理）
- [ ] `NocSystem`（6 組 API）
- [ ] `MetricsCollector` 統計收集
- [ ] 單元測試 (GoogleTest)

### Phase 2: Verification & Golden Reference

**目標**: 建立驗證框架 + I/O pattern 支援。

- [ ] `GoldenManager`（Write capture + Read verification）
- [ ] `HostMemory` / `LocalMemory`
- [ ] I/O pattern 載入/產出（Config JSON, Memory HEX, Traffic JSON）
- [ ] Response Log + Memory State dump
- [ ] Theory Validator（Throughput/Latency bounds）
- [ ] Consistency Validator（Little's Law, Flit Conservation）
- [ ] Batch performance test runner

### Phase 3: Co-Simulation Bridge

**目標**: 實作 DPI-C bridge，支援 RTL co-simulation。

- [ ] `noc_dpi.cpp` — DPI-C 函數匯出（thin wrapper）
- [ ] `RtlRouterProxy` / `RtlNIProxy`（hot-swap 實作）
- [ ] RTL harness (`noc_harness.sv`)
- [ ] 8-Phase tick 同步機制（C++ ↔ RTL）
- [ ] Co-simulation 測試（Questa / Verilator）

### Phase 4: Advanced Features

**目標**: 擴展進階功能。

- [ ] QoS 支援（Priority-based arbitration）
- [ ] Multicast（Rectangle Multicast via RCR）
- [ ] Credit-Based Flow Control (Version B)
- [ ] Virtual Channel 支援
- [ ] Cycle Trace 輸出（VCD / JSON）
- [ ] 效能分析工具（hotspot visualization）

---

## 7. 品質要求

| 項目 | 標準 |
|------|------|
| **語言** | C++17（相容 gcc/clang/MSVC） |
| **Build** | CMake 3.16+ |
| **測試** | GoogleTest, 目標覆蓋率 80%+ |
| **文件** | Doxygen 註解於 public API |
| **風格** | clang-format (Google style 或自訂) |
| **靜態分析** | clang-tidy, cppcheck |
| **移植性** | Linux + Windows (MSYS2/MinGW) |

---

## 8. 參考資源

| 資源 | 說明 |
|------|------|
| `docs/design/` | NoC 架構設計文件 |
| `docs/images/noc_env.txt` | 系統架構圖（ASCII） |
| `examples/patterns/` | I/O pattern 範例檔案 |
| `docs/ref/IHI0050H_amba_chi_architecture_spec.pdf` | AMBA CHI 規格 |
| `docs/ref/AMBAaxi.pdf` | AMBA AXI 規格 |
| `docs/ref/Arteris_IP_Overview_20210820_1_HotChips.pdf` | Arteris NoC IP 參考 |
| `docs/ref/Ch15_NoC_Design.pdf` | NoC 設計教材 |
