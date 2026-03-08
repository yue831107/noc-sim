# NoC C++ Model - Project Goals

## Executive Summary

本專案建立 NoC (Network-on-Chip) 的 **C++ Cycle-Accurate Behavior Model**，同時作為 **Pre-silicon 效能評估平台** 與 **RTL Co-simulation Golden Reference**。

同一組 Input Pattern 餵入 C++ model 和 RTL，C++ 的 Output 作為 Golden，與 RTL Output 進行 byte-exact / cycle-accurate 比對。

> **Key Takeaway**: 一個 model，兩種用途 — 效能評估與 RTL 驗證共用同一套 cycle-accurate 引擎。

---

## Key Numbers

| 項目 | 數值 |
|------|------|
| Mesh 預設大小 | 4 × 4（16 nodes，最大 16×16 = 256） |
| Flit 寬度 | 408-bit（Header 56b + Payload 352b） |
| Physical Link | 每 pair 4 × 410 = 1,640 bits（Req/Rsp × fwd/rev） |
| Simulation Phases | 8 phases per cycle |
| API Groups | 6 組（A~F） |
| Internal Components | 十餘個（Router, NI, Channel, Memory 等） |
| Flow Control Modes | 2（Valid/Ready, Credit-Based） |
| QoS Levels | 16 |

---

## 1. 專案背景

本專案目標是建立一套 NoC 的 C++ Behavior Model，架構規格如下：

- **Topology**: 4×4 Mesh（Uniform Router，每個 Router 可擁有 0~4 個 LOCAL port）
- **Routing**: XY Routing + Wormhole Switching（deadlock-free）
- **Flow Control**: 雙模式 — Valid/Ready (Version A) 和 Credit-Based (Version B)，compile-time 選擇
- **Interface**: AXI4 (AW/W/AR/B/R 五通道)
- **Physical Channel**: Req/Rsp 物理分離（雙軌架構）
- **Flit**: 固定 408-bit（Header 56b + Payload 352b）
- **Injection Mode**: `HOST_DMA`（集中從 gateway node 注入）或 `DIRECT_PE`（分散到各 PE 直接注入）

### Node 結構

每個 node 由 PE（Processing Element）+ NI + Router 組成。PE 含 DMA 與 local DRAM，透過 AXI 介面連接 NI。每個 PE 的 DMA 可發起跨 node transaction（NUMA-like access）。

可配置的 **Gateway Node** 擁有額外的 LOCAL port（`port_id=1`），連接外部 Host DMA，使 Router 至少使用 2 個 LOCAL port。

> **Key Takeaway**: 每個 node = PE + NI + Router；gateway node 多一個 Host DMA 入口。

---

## 2. 核心目標

### 目標 A: 硬體前效能評估 (Pre-Silicon Performance Evaluation)

在 RTL 實作之前，以 C++ cycle-accurate model 進行 traffic pattern 掃描與參數空間探索。

| 需求 | 說明 |
|------|------|
| **Cycle-Accurate 模擬** | C++ model，模擬速度高於 RTL simulation |
| **參數掃描** | mesh size、buffer depth、routing algorithm、pipeline delay 等 |
| **Traffic Pattern** | neighbor, shuffle, bit_reverse, random, transpose 等 |
| **統計收集** | Throughput, Latency, Buffer Utilization, Flit Conservation |
| **批量測試** | 自動化效能測試，輸出 JSON 報告 |
| **可視化** | 輸出資料供外部工具繪圖 |

**預期產出**: 效能比較報告、瓶頸分析、QoS 策略評估、RTL 最佳參數建議。

> **Key Takeaway**: 在 RTL 之前，用 C++ model 完成設計空間探索。

---

### 目標 B: RTL 共模驗證 (RTL Co-Simulation / Golden Reference)

C++ model 作為 RTL 的 golden reference，確保硬體實作與行為模型一致。

| 需求 | 說明 |
|------|------|
| **Cycle-Accurate** | 與 RTL 逐 cycle 比對，結果完全一致 |
| **DPI-C 介面** | DPI-C Bridge 每 cycle 與 RTL module 交換 port 信號（flit + valid + ready + credit） |
| **Hot-Swap** | 單一 Router 或 NI 可替換為 RTL，其餘維持 C++ |
| **Response 驗證** | 攔截 RTL response，與 C++ model 預期結果比對 |
| **Data Integrity** | Golden Data 機制，支援 MULTICAST |
| **Waveform Dump** | 可選擇輸出 C++ model 內部狀態供 waveform 分析 |

完整架構圖見 [`docs/images/noc_architecture.svg`](images/noc_architecture.svg)。

> **Key Takeaway**: C++ model = Golden Reference，支援 per-instance 替換 RTL 進行漸進驗證。

---

## 3. 架構設計

### 3.1 Software / Hardware Architecture

| 區塊 | 名稱 | 職責 |
|------|------|------|
| **User Code** | User Code | 使用者撰寫的 C++ test program 或 Python script |
| **Public API** | NocSystem Public API | 6 組 API（`noc_api.h`），詳見 §4.2 |
| **Internal** | Internal Components | TrafficManager / Mesh / Router / NI / Channel / Memory / Stats + **HOT-SWAP BOUNDARY** |
| **Co-Sim** | Co-Sim Bridge + RTL | DPI-C（每 cycle port 信號交換）+ SystemVerilog RTL modules |

Internal Components 完全不依賴 simulator，可獨立編譯為 library。

> **Key Takeaway**: User Code 透過 Public API 操作 Internal Components；Co-Sim Bridge 負責 C++ ↔ RTL 信號交換。

### 3.2 Abstract Interface + Hot-Swap

透過 `Router_Interface<Mode>` 和 `NI_Interface<Mode>` 抽象介面，支援：
- **CppRouter / CppNI**: 純 C++ 行為模型（用於獨立效能模擬）
- **RouterDpiBridge / NIDpiBridge**: 透過 DPI-C 橋接到 RTL（用於 co-simulation）
- 可 per-instance 替換：例如 15 個 CppRouter + 1 個 RouterDpiBridge

Flow control mode 為 compile-time template parameter，不產生未使用的信號：
- `PortOutputVR`：valid + ready，無 credit
- `PortOutputCredit`：credit return，無 ready

> **Key Takeaway**: 抽象介面 = 可插拔邊界，C++ 與 RTL 實作共用同一合約。

### 3.3 Channel-Based Interconnect

Router 之間、NI 與 Router 之間使用 `Channel<T>` 連接：
- **latency=1**（預設）：下一 cycle 可見，等同 pipeline register
- **latency=0**：combinational wire
- **latency>1**：模擬 long-wire 或額外 pipeline stage

### 3.4 Pluggable Components (Factory Pattern)

| 元件 | Abstract Base | 可選實作 |
|------|-------------|---------|
| VC Allocator | `Allocator` | RoundRobin, iSLIP |
| SW Allocator | `Allocator` | QoSAwareRR, iSLIP |
| Traffic Pattern | `TrafficPattern` | Uniform, Tornado, Custom |
| Buffer Policy | `BufferPolicy` | Private, Shared |

### 3.5 Completion 查詢模式

Transaction 提交後（`submit_write()` / `submit_read()`），使用者拿到一個 `TxnHandle`。每個 cycle，TrafficManager 的 `tick()` 會檢查 NMU 是否收到 AXI response，收到就更新內部狀態表。使用者取得結果的方式有兩種：

| 模式 | 做法 | 類比 |
|------|------|------|
| **Polling** | 使用者主動呼叫 `is_complete(handle)` 查詢狀態表 | 類似 testbench 中輪詢 `bvalid` |
| **Callback** | 使用者預先註冊 function，狀態表更新時自動通知 | 類似 UVM event trigger |

兩者可同時使用。詳見 [Simulation Platform](docs/design/09_simulation.md)。

### 3.6 C++ 與 RTL 分工

| 負責方 | 職責 |
|--------|------|
| **C++** | Routing 決策、Flow Control、Flit 組裝/解析、Arbitration、Buffer 管理 |
| **RTL** | 物理通道傳輸、時序對齊、信號驅動 |
| **DPI-C** | 薄層橋接 — DPI-C Bridge 每 cycle 呼叫 `noc_set_port_input()` / `noc_get_port_output()` 交換 port 信號 |

> **Key Takeaway**: C++ 做邏輯，RTL 做信號，DPI-C 只搬資料。

---

## 4. 系統架構

完整架構圖見 [`docs/images/noc_architecture.svg`](images/noc_architecture.svg)。

### 4.1 I/O Pattern 定義

| Pattern | 方向 | 格式 | 用途 | Golden 比對 |
|---------|------|------|------|:----------:|
| Config | INPUT | `.json` | 系統參數（mesh size, buffer depth, pipeline delay 等） | — |
| Memory Init | INPUT | `.hex` | 初始記憶體（HostMemory + 各 node LocalMemory） | — |
| Traffic | INPUT | `.json` + `.hex` | 交易序列 + write data | — |
| Memory State | OUTPUT | `.hex` | 最終記憶體 dump | byte-exact |
| Response Log | OUTPUT | `.json` + `.hex` | 交易回應 + read data | cycle-accurate |
| Cycle Trace | OUTPUT | `.vcd` / `.json` | 時序追蹤（debug only） | — |
| Statistics | OUTPUT | `.json` | 效能統計（derived） | — |

範例 pattern 檔案見 `examples/patterns/`。

> **Key Takeaway**: 3 種 Input + 4 種 Output，其中 Memory State 與 Response Log 用於 Golden 比對。

### 4.2 NocSystem Public API（6 組）

| Group | 名稱 | 主要 API |
|-------|------|---------|
| **A** | Construction | `NocSystem(config)`, `load_memory()`, `load_traffic()` |
| **B** | Transaction | `submit_write()`, `submit_read()`, `submit_multicast_write()` → `TxnHandle` |
| **C** | Simulation | `process_cycle()`, `run(N)`, `run_until_idle()`, `run_all()` |
| **D** | Completion | `is_complete()`, `get_status()`, `get_read_data()`, `set_completion_callback()` |
| **E** | Metrics | `get_metrics()`, `verify()` |
| **F** | Debug & Output | `get_router()`, `get_ni()`, `dump_state()`, `generate_golden()` |

**Convenience API** — 載入、執行、產出：

```cpp
static NocConfig load_config(const std::string& json_path);
void load_memory(const std::string& hex_dir);
void load_traffic(const std::string& json_path);
void run_all();
void generate_golden(const std::string& output_dir) const;
```

> **Key Takeaway**: 6 組 API 覆蓋完整 lifecycle — 建構 → 提交 → 執行 → 查詢 → 驗證 → Debug。

### 4.3 Internal Components

| 元件 | 職責 |
|------|------|
| `NocSystem` | 頂層系統，協調所有元件，暴露 6 組 API |
| `TrafficManager` | 模擬 Host DMA / PE DMA 行為，管理 AXI transaction lifecycle，支援 `HOST_DMA` / `DIRECT_PE` 雙模式注入 |
| `Mesh` | 拓撲組裝 + 連線管理 + 8-Phase cycle 驅動 |
| `CppRouter` | 6-stage pipeline: InputQueuing → RouteEvaluate → VCAllocEvaluate → SWAllocEvaluate → SwitchTraverse → OutputQueuing |
| `CppNI` | NMU（AXI → flit 轉換 + ECC/QoS）+ NSU（flit → AXI + Memory 操作 + burst reassembly） |
| `Channel<T>` | 帶可配置延遲的 link model |
| `Allocator` | 抽象 arbiter base → RoundRobin / iSLIP / QoSAwareRR |
| `InputBuffer` / `BufferState` | Data storage 與 credit tracking 分離 |
| `Flit` | 408-bit flit 結構 + object pooling |
| `RouteCompute` | XY routing + multicast RCR |
| `Scoreboard` | Online expected vs actual tracking (per-txn) |
| `MetricsCollector` | Per-node, per-class 效能統計 |
| `HostMemory` / `LocalMemory` | 記憶體模型 |

### 4.4 Replaceable Boundary 總覽

**Hot-Swap（per-instance 替換）**:

| 元件 | Abstract Interface | C++ Model | DPI-C Bridge |
|------|-------------------|-----------|-------------|
| Router | `Router_Interface<Mode>` | `CppRouter` | `RouterDpiBridge` |
| NI | `NI_Interface<Mode>` | `CppNI` | `NIDpiBridge` |

**Pluggable（Config string 選擇）**: VC Allocator, SW Allocator, Traffic Pattern, Buffer Policy

**固定 C++ 實作**: TrafficManager, Mesh, Channel, Scoreboard, MetricsCollector, HostMemory/LocalMemory

> **Key Takeaway**: Router 和 NI 可 Hot-Swap 到 RTL；Allocator 等可 config 切換；其餘固定。

### 4.5 DPI-C Bridge

DPI-C Bridge 每 cycle 透過 DPI-C 與 RTL module 交換 port 信號（flit + valid + ready + credit）：

```cpp
extern "C" {
    // 每 cycle 將 C++ 側信號送入 RTL module 的 port
    void noc_set_port_input(int node, int port, int ch,
                            int valid, const uint8_t* flit, int credit);

    // 每 cycle 將 RTL module 的 port 輸出讀回 C++
    void noc_get_port_output(int node, int port, int ch,
                             int* valid, uint8_t* flit, int* credit);
}
```

> **Key Takeaway**: DPI-C 只做 port 信號搬運，一個 cycle 一次，保持薄層。

### 4.6 8-Phase Cycle Model

每個 simulation cycle 依序執行 8 個 phase，將 RTL 的並行行為拆為循序操作：

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

> **Key Takeaway**: 8 phase = RTL 一個 clock cycle 的循序展開，保證因果正確。

---

## 5. 目錄結構

```
noc_c_model/
├── docs/                       # 設計文件
│   ├── design/                 # 架構規格
│   └── images/                 # 圖片資源（含 noc_architecture.svg）
├── include/                    # Public headers
│   └── noc/
│       ├── noc_api.h           # 使用者 API（6 組）
│       ├── types.h             # 共用型別定義
│       └── config.h            # NocConfig
├── src/                        # C++ 源碼
│   ├── core/                   # Router, NI, Flit, Buffer, Arbiter, Routing, Mesh
│   ├── memory/                 # LocalMemory, HostMemory
│   ├── system/                 # NocSystem, TrafficManager, Scoreboard
│   ├── stats/                  # MetricsCollector
│   └── bridge/                 # DPI-C simulator bridge
├── rtl/                        # RTL Harness (SystemVerilog)
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

| Phase | 名稱 | 目標 | 關鍵交付物 |
|-------|------|------|-----------|
| **1** | Core Library | 獨立 C++ model，可執行效能模擬 | Flit, Channel, Router, NI, Mesh, NocSystem, TrafficManager |
| **2** | Verification | 驗證框架 + I/O pattern 支援 | Scoreboard, Memory, I/O loader/dumper, Validator |
| **3** | Co-Sim Bridge | DPI-C bridge，RTL co-simulation | DPI-C Bridge, RTL harness, 同步機制 |
| **4** | Advanced | 進階功能 | QoS, Multicast, Credit Flow Control, Virtual Channel |

### Phase 1: Core Library

- [ ] 基礎資料結構: `Flit`（408-bit）、`AXITransaction`、`NocConfig`
- [ ] `Channel<T>` — 帶可配置延遲的 link model
- [ ] `InputBuffer` + `BufferState`（data/credit 分離）
- [ ] `Allocator` 抽象 + RoundRobin / iSLIP 實作
- [ ] `CppRouter`（6-stage pipeline）
- [ ] `CppNI`（NMU: AXI → flit + NSU: flit → AXI + memory）
- [ ] `Mesh` topology 組裝 + 8-Phase cycle model
- [ ] `TrafficManager`（AXI transaction lifecycle + 雙模式注入）
- [ ] `NocSystem`（6 組 API）
- [ ] `MetricsCollector` 統計收集
- [ ] 單元測試 (GoogleTest)

### Phase 2: Verification & Golden Reference

- [ ] `Scoreboard`（Online expected vs actual per-txn tracking）
- [ ] `HostMemory` / `LocalMemory`
- [ ] I/O pattern 載入/產出（Config JSON, Memory HEX, Traffic JSON）
- [ ] Response Log + Memory State dump
- [ ] Theory Validator（Throughput/Latency bounds）
- [ ] Consistency Validator（Little's Law, Flit Conservation）
- [ ] Batch performance test runner

### Phase 3: Co-Simulation Bridge

- [ ] `noc_dpi.cpp` — DPI-C 函數匯出（`noc_set_port_input` / `noc_get_port_output`）
- [ ] `RouterDpiBridge` / `NIDpiBridge`（hot-swap 實作）
- [ ] RTL harness (`noc_harness.sv`)
- [ ] 8-Phase tick 同步機制（C++ ↔ RTL）
- [ ] Co-simulation 測試（Questa / Verilator）

### Phase 4: Advanced Features

- [ ] QoS 支援（16-level priority + QoS-Aware RR arbitration）
- [ ] Multicast（Rectangle Multicast via RCR + in-network B reduction）
- [ ] Credit-Based Flow Control (Version B)
- [ ] Virtual Channel 支援
- [ ] Cycle Trace 輸出（VCD / JSON）
- [ ] 效能分析工具（hotspot visualization）

> **Key Takeaway**: 4 個 Phase 漸進交付 — 先能跑、再能驗、接上 RTL、最後加功能。

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
| `docs/design/` | NoC 架構設計文件（含 `00_architecture.md` 入口文件） |
| `docs/images/noc_architecture.svg` | 系統架構圖（4 欄式，含所有元件與連線） |
| `examples/patterns/` | I/O pattern 範例檔案 |
| `docs/ref/IHI0050H_amba_chi_architecture_spec.pdf` | AMBA CHI 規格 |
| `docs/ref/AMBAaxi.pdf` | AMBA AXI 規格 |
| `docs/ref/Arteris_IP_Overview_20210820_1_HotChips.pdf` | Arteris NoC IP 參考 |
| `docs/ref/Ch15_NoC_Design.pdf` | NoC 設計教材 |
