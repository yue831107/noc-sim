# 內部介面架構

本文件定義 NoC 元件之間的 physical link 信號、flow control 機制、連接拓撲與 cycle processing model。所有參數以 [Flit Format](04_flit.md) 為基準，設計目標為可直接映射至 C++ cycle-accurate model 與 RTL 實作。

---

## 1. Physical Link 定義

每條 physical link 承載單方向 flit 傳輸。Request 與 Response 使用獨立 physical link（full-duplex），消除 request-response circular dependency。

### 1.1 Link Signal Table

與 [04_flit.md Section 4.1](04_flit.md) 完全一致：

| Signal | Bit | Width | Direction | Description |
|--------|-----|-------|-----------|-------------|
| `valid` | [0] | 1 | Sender → Receiver | Flit data valid |
| `ready` | [1] | 1 | Receiver → Sender | Receiver can accept |
| `flit` | [409:2] | 408 | Sender → Receiver | Flit data (header + payload) |
| | | **410** | | **Total physical link width** |

- 兩版 header（Version A / Version B）的 physical link 寬度相同 — version 差異僅在 header 內部 bit [55:50] 的解釋，不影響 wire width。
- 每個 Router port 具有一對 link：ingress（接收）與 egress（發送），合計 820 bits per port pair。
- Req/Rsp 雙 physical network 每個 Router 方向需 2 × 820 = 1,640 bits（不含 clock/reset）。

### 1.2 C++ 結構定義

```cpp
// Physical link — one direction, 410 bits
struct PhysicalLink {
    bool     valid;              // 1 bit: sender asserts when flit is valid
    bool     ready;              // 1 bit: receiver asserts when buffer has space
    uint8_t  flit[51];           // 408 bits (51 bytes): header(56b) + payload(352b)
};

static constexpr int FLIT_WIDTH         = 408;  // bits
static constexpr int PHYSICAL_LINK_WIDTH = 410;  // valid(1) + ready(1) + flit(408)
```

### 1.3 Handshake 基本規則

無論 Version A 或 Version B，physical link 層的傳輸規則相同：

1. Sender 在資料準備好時設 `valid = 1` 並驅動 `flit`
2. Receiver 在 buffer 有空間時設 `ready = 1`
3. **Transfer 發生條件：`valid && ready` 同時為 1**（同一 cycle）
4. Sender 在 transfer 完成後的下一 cycle 可撤銷 `valid`（或維持以連續傳送）
5. **`valid` 一旦 assert 不得在未被 accept 前撤銷**（AXI-style stable requirement）
6. **`ready` 可隨時變動**（back-pressure）

---

## 2. Flow Control 架構

04_flit.md 定義了兩種 flow control 版本，差異在於 Virtual Channel (VC) 的實現方式。以下明確定義兩者的信號、行為與適用場景。

### 2.1 Version A — Valid/Ready Handshake（Per-VC Physical Lines）

**概念：** 每個 VC 擁有獨立的 physical link（valid + ready + flit），VC 選擇在 signal level 完成。

```
                         VC 0: valid/ready/flit (410 bits)
Sender ──────────────────────────────────────────────────► Receiver
                         VC 1: valid/ready/flit (410 bits)
       ──────────────────────────────────────────────────►
                         VC 2: valid/ready/flit (410 bits)
       ──────────────────────────────────────────────────►
                                   ...
                         VC N-1: valid/ready/flit (410 bits)
       ──────────────────────────────────────────────────►
```

**信號定義：**

| Signal per VC | Width | Description |
|---------------|-------|-------------|
| `valid[vc]` | 1 | Sender asserts for this VC |
| `ready[vc]` | 1 | Receiver has buffer space for this VC |
| `flit[vc]` | 408 | Flit data（header 中 bit [55:50] 全為 rsvd，無 vc_id） |

**特性：**

| Property | Value |
|----------|-------|
| Wires per direction | NumVC x 410 |
| Wire count (4 VC) | 1,640 wires/dir |
| Wire count (8 VC) | 3,280 wires/dir |
| Header `vc_id` | 不使用（bit [55:50] = rsvd 6b） |
| Arbitration | Receiver 端 per-VC 獨立 accept |
| Flow control | Per-VC `ready` 直接反映 buffer 狀態 |

**C++ 結構：**

```cpp
static constexpr int NUM_VC = 4;  // Version A: 通常 VC <= 2

// Version A: Per-VC physical lines
// 每個 VC 擁有獨立的 valid/flit/ready 信號
struct PortOutputVR {
    bool valid[NUM_VC] = {};       // per-VC: outgoing flit valid
    Flit flit[NUM_VC]  = {};       // per-VC: outgoing flit data (408 bits each)
    bool ready[NUM_VC] = {};       // per-VC: can accept incoming flit
};
// Total wires per direction: NUM_VC × (1 + 408 + 1) = NUM_VC × 410 bits
```

### 2.2 Version B — Credit-Based Flow Control（Header VC ID）

**概念：** 所有 VC 共享單一 physical link（409 bits），VC ID 編碼於 header bit [52:50]。Receiver 透過 credit return 信號通知 sender buffer 可用空間。

```
                    Shared link: valid/flit (409 bits)
Sender ──────────────────────────────────────────────────► Receiver
                    Credit return: credit[NUM_VC] (per-VC independent bits)
       ◄──────────────────────────────────────────────────
```

**Forward path 信號（sender → receiver）：**

| Signal | Width | Description |
|--------|-------|-------------|
| `valid` | 1 | Flit valid（sender 有 flit 且 credit[vc_id] > 0） |
| `flit` | 408 | Flit data（header bit [52:50] = vc_id） |

**Credit return 信號（receiver → sender）：**

| Signal | Width | Description |
|--------|-------|-------------|
| `credit[NUM_VC]` | NUM_VC | Per-VC credit return pulse（每 VC 獨立歸還） |

> **注意：** 採用 per-VC 獨立 credit bit，而非 `credit_valid + credit_vc_id` 的序列化方案。多個 VC 可在同一 cycle 同時歸還 credit。

**特性：**

| Property | Value |
|----------|-------|
| Forward wires per dir | 409 (valid + flit) |
| Credit return wires | NUM_VC (per-VC credit pulse) |
| Total wires per dir | 409 + NUM_VC |
| Header `vc_id` | 3 bits (bit [52:50]) |
| Arbitration | Sender 端 mux（選擇哪個 VC 的 flit 發送） |
| Flow control | Per-VC credit counter |

**Credit Counter 精確行為：**

```
Initial state:
  credit[vc] = BUFFER_DEPTH_PER_VC    // 例: buffer_depth=4, NUM_VC=4 → 每 VC 1 entry
                                       // 或 buffer_depth=16, NUM_VC=4 → 每 VC 4 entries

Send condition:
  can_send(vc) = (credit[vc] > 0) && valid && ready

On send (valid && ready, flit.header.vc_id == vc):
  credit[vc] -= 1                      // Sender 消耗一個 credit

On credit return (credit[vc] pulse):
  credit[vc] += 1                      // Receiver 歸還一個 credit（多 VC 可同時歸還）

Invariant:
  0 <= credit[vc] <= BUFFER_DEPTH_PER_VC
  credit[vc] == (downstream buffer free slots for this VC)
```

**C++ 結構：**

```cpp
static constexpr int MAX_VC = 8;  // Version B: 最多 8 VC

// Version B: Credit-based, shared physical link
// 所有 VC 共享一條 forward link，credit return per-VC 獨立
struct PortOutputCredit {
    // Forward path (sender → receiver)
    bool    valid = false;             // shared: outgoing flit valid
    Flit    flit{};                    // shared: flit data (header bit [52:50] = vc_id)

    // Credit return (receiver → sender) — per-VC independent
    bool    credit[MAX_VC] = {};       // per-VC: credit return pulse
};
// Total wires per direction: (1 + 408) + MAX_VC = 409 + MAX_VC bits

// Per-VC credit counter (maintained at sender side)
struct CreditCounter {
    int credits[MAX_VC];               // Per-VC available credits

    void init(int buffer_depth_per_vc) {
        for (int i = 0; i < MAX_VC; i++)
            credits[i] = buffer_depth_per_vc;
    }

    bool can_send(int vc) const { return credits[vc] > 0; }
    void consume(int vc)        { credits[vc]--; }      // On flit send
    void release(int vc)        { credits[vc]++; }      // On credit return
};
```

### 2.3 Version 選擇準則

| Criteria | Version A (Valid/Ready) | Version B (Credit-Based) |
|----------|------------------------|--------------------------|
| **VC 數量** | VC <= 2 | VC >= 2（推薦 VC >= 4） |
| **Wire 成本** | 高（線性增長） | 低（409 + NUM_VC wires） |
| **Header overhead** | 0 bits（無 vc_id） | 3 bits（from rsvd） |
| **Sender 複雜度** | 低（直接驅動對應 VC link） | 中（credit tracking + VC mux） |
| **Receiver 複雜度** | 低（per-VC ready 直接反映 buffer） | 中（credit return 邏輯） |
| **Timing** | ready 為 combinational feedback | Credit 為 registered（1-cycle return latency） |
| **適用場景** | 簡化驗證、低 VC 數、FPGA prototype | ASIC tape-out、高 VC 數、wire-constrained |

**建議：** ASIC 實作推薦 Version B（credit-based）。Version A 適用 FPGA prototype 或 VC <= 2 的簡化設計。本 C++ model 同時支援兩者，由 compile-time configuration 選擇。

---

## 3. Abstract Interface

透過 abstract interface 實現 C++ model 與 RTL 的 hot-swap。Router 和 NI 均實作相同的介面合約，Simulation Driver 透過 wiring loop 連接所有元件，無需區分 C++ 或 RTL 實作。

### 3.1 設計原則

```
┌─────────────────────────────────────────────────────────────┐
│                   Co-Simulation Platform                      │
│                                                             │
│  ┌──────────────┐      Abstract       ┌──────────────┐     │
│  │  CppRouter   │──── Interface ──────│  CppNI       │     │
│  │  (C++ Model) │      Contract       │  (C++ Model) │     │
│  └──────────────┘         │           └──────────────┘     │
│         ▲                 │                  ▲              │
│         │ implements      │                  │ implements   │
│         ▼                 ▼                  ▼              │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────┐        │
│  │Router_       │  │Simulation │  │NI_           │        │
│  │Interface<M>  │◄─┤Driver     ├─►│Interface<M>  │        │
│  └──────────────┘  │(Wiring)   │  └──────────────┘        │
│         ▲          └───────────┘         ▲              │
│         │ implements                     │ implements   │
│         ▼                                ▼              │
│  ┌──────────────┐                 ┌──────────────┐        │
│  │RtlRouterProxy│                 │RtlNIProxy    │        │
│  │  (DPI-C)     │                 │  (DPI-C)     │        │
│  └──────────────┘                 └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**核心規則：**

1. **Interface 是合約** — `Router_Interface` 和 `NI_Interface` 定義唯一的元件間通訊方式
2. **Flow control mode 是 compile-time 參數** — 整個系統統一使用同一種 mode
3. **不帶多餘信號** — Valid/Ready mode 無 credit 信號；Credit mode 無 ready 信號
4. **NI-Router 與 Router-Router 使用相同介面** — 不做區分

### 3.2 Abstract Interface 定義

```cpp
namespace noc {

enum class Channel { REQ, RSP };
enum class FlowControlMode { VALID_READY, CREDIT };

// PortOutput<Mode> 定義見 Section 2:
//   FlowControlMode::VALID_READY → PortOutputVR
//   FlowControlMode::CREDIT      → PortOutputCredit

template<FlowControlMode Mode>
using PortOutput = std::conditional_t<
    Mode == FlowControlMode::VALID_READY,
    PortOutputVR,
    PortOutputCredit
>;

// ============================================
// Router Abstract Interface
// ============================================
template<FlowControlMode Mode>
class Router_Interface {
public:
    virtual ~Router_Interface() = default;

    /// Port 數量 (mesh ports + local ports)
    virtual int num_ports() const = 0;

    /// 取得指定 port 的 output 信號（本元件驅動的信號）
    /// - forward: valid + flit（本元件送出的 flit）
    /// - backward: ready (VR) 或 credit (Credit)（本元件接收能力）
    virtual const PortOutput<Mode>& get_output(int port, Channel ch) const = 0;

    /// 設定指定 port 的 input 信號（對端元件驅動的信號）
    virtual void set_input(int port, Channel ch, const PortOutput<Mode>& in) = 0;

    /// 推進一個 simulation cycle
    virtual void tick() = 0;
};

// ============================================
// NI Abstract Interface
// ============================================
template<FlowControlMode Mode>
class NI_Interface {
public:
    virtual ~NI_Interface() = default;

    /// NI 面向 Router 的 output 信號
    virtual const PortOutput<Mode>& get_output(Channel ch) const = 0;

    /// 設定 NI 的 input 信號（來自 Router）
    virtual void set_input(Channel ch, const PortOutput<Mode>& in) = 0;

    /// 推進一個 simulation cycle
    virtual void tick() = 0;
};

}  // namespace noc
```

### 3.3 PortOutput 信號方向

每個 port 的 `PortOutput` 包含該元件**主動驅動**的所有信號，方向上是對稱的：

```
Component A                                    Component B
get_output(port_a)                             get_output(port_b)
┌──────────────────┐                           ┌──────────────────┐
│ valid  (A→B flit)│──────── wire ────────────►│ 被 set_input 接收 │
│ flit   (A→B data)│──────── wire ────────────►│                  │
│ ready  (A 接收力) │──────── wire ────────────►│ (VR: A 可收 B 的) │
│  or                                           │                  │
│ credit (A 歸還)  │──────── wire ────────────►│ (Cr: A 消耗通知)  │
└──────────────────┘                           └──────────────────┘

┌──────────────────┐                           ┌──────────────────┐
│ 被 set_input 接收 │◄──────── wire ───────────│ valid  (B→A flit)│
│                  │◄──────── wire ───────────│ flit   (B→A data)│
│                  │◄──────── wire ───────────│ ready/credit     │
└──────────────────┘                           └──────────────────┘
```

### 3.4 Simulation Driver Wiring

Simulation Driver（Mesh）負責每 cycle 讀取所有元件的 output 並設定對端的 input：

```cpp
/// 連接描述
struct Connection {
    Router_Interface<Mode>* a;    int port_a;     // A 側 port index
    Router_Interface<Mode>* b;    int port_b;     // B 側 port index
};

struct NIConnection {
    NI_Interface<Mode>*     ni;
    Router_Interface<Mode>* router;  int local_port;
};

/// Wiring loop — 每 cycle 執行一次
template<FlowControlMode Mode>
void Mesh<Mode>::wire_all() {
    // Router ↔ Router
    for (auto& conn : router_connections) {
        for (auto ch : {Channel::REQ, Channel::RSP}) {
            auto& out_a = conn.a->get_output(conn.port_a, ch);
            auto& out_b = conn.b->get_output(conn.port_b, ch);
            conn.b->set_input(conn.port_b, ch, out_a);  // A→B
            conn.a->set_input(conn.port_a, ch, out_b);  // B→A
        }
    }
    // NI ↔ Router (LOCAL port) — 同一機制
    for (auto& conn : ni_connections) {
        for (auto ch : {Channel::REQ, Channel::RSP}) {
            auto& out_ni = conn.ni->get_output(ch);
            auto& out_r  = conn.router->get_output(conn.local_port, ch);
            conn.router->set_input(conn.local_port, ch, out_ni);  // NI→Router
            conn.ni->set_input(ch, out_r);                        // Router→NI
        }
    }
}
```

### 3.5 Hot-Swap 機制

Flow control mode 為 compile-time 參數，所有元件使用相同 mode。Hot-swap 在同一 mode 下替換 C++ ↔ RTL 實作：

```cpp
// Pure C++ simulation
auto router_00 = std::make_unique<CppRouter<Mode>>(Coord{0,0}, config);
auto ni_00     = std::make_unique<CppNI<Mode>>(config);

// Hot-swap: 將 Router (2,1) 替換為 RTL proxy
auto router_21 = std::make_unique<RtlRouterProxy<Mode>>("router_2_1", dpi_handle);

// Mesh 不需修改 — 只要實作 Router_Interface<Mode> 即可
mesh.routers[{2,1}] = std::move(router_21);
```

| Substitution Mode | Router | NI | Use Case |
|-------------------|--------|----|----------|
| Pure C++ | CppRouter | CppNI | 高速效能模擬、參數掃描 |
| NI RTL | CppRouter | RtlNIProxy | NI RTL 驗證 |
| Router RTL | RtlRouterProxy | CppNI | Router RTL 驗證 |
| Full RTL | RtlRouterProxy | RtlNIProxy | 系統級 RTL co-sim |

### 3.6 Latency Model

| Model | Behavior | RTL Mapping | Use Case |
|-------|----------|-------------|----------|
| **Zero-cycle** | `wire_all()` 在同 cycle 內傳遞信號 | Wire / combinational path | 預設模型 |
| **1-cycle** | 信號延遲一個 cycle 才被對端收到 | Pipeline register on link | Long-wire timing closure |

**預設採用 zero-cycle propagation** — `wire_all()` 設定的信號在下一 cycle 的 `tick()` 即可被對端處理。等同 RTL wire 直連。若需 1-cycle latency，可在 Connection 內加入 staging register。

---

## 4. CppRouter 內部實作

本節描述 `CppRouter`（C++ model 的 Router 實作）的內部結構。此為 `Router_Interface<Mode>` 的具體實作，RtlRouterProxy 有不同的內部實作但對外提供相同介面。

### 4.1 內部 Port 結構

CppRouter 內部每個 port 維護 input/output 信號與 buffer 狀態：

```cpp
template<FlowControlMode Mode>
struct InternalPort {
    // === 來自 set_input() 的信號（對端驅動）===
    PortOutput<Mode> input{};

    // === 供 get_output() 回傳的信號（本端驅動）===
    PortOutput<Mode> output{};

    // === Internal State ===
    FlitBuffer input_buffer;     // Input buffer (depth = config.buffer_depth)

    // === Version B only ===
    CreditCounter output_credits; // Per-VC credit counters for egress (sender side)
};
```

### 4.2 tick() 內部處理

CppRouter 的 `tick()` 實作 8-phase cycle model（見 Section 7），概要如下：

| Step | 對應 Phase | Description |
|------|-----------|-------------|
| Sample | Phase 1 | 從 `input` 讀取 flit → push 到 `input_buffer` |
| Update ready/credit | Phase 3/7 | VR: `output.ready = !input_buffer.full()`; Credit: 設定 `output.credit[vc]` |
| Route & Forward | Phase 4 | 從 `input_buffer` peek → XY routing → 設定目標 port 的 `output.valid/flit` |
| Clear accepted | Phase 6 | 根據 `input` 中對端的 ready/credit 確認 handshake 完成，清除 `output` |

---

## 5. NI-Router 連接

### 5.1 統一介面原則

NI 與 Router 的連接使用與 Router-Router **完全相同**的 abstract interface 和 wiring 機制。NI 實作 `NI_Interface<Mode>`，Router 實作 `Router_Interface<Mode>`，Simulation Driver 透過 `wire_all()` 統一處理。

```
NI_Interface<Mode>                         Router_Interface<Mode>
==================                         =====================
get_output(REQ)  ──── wire_all() ────────► set_input(local_port, REQ)
set_input(REQ)   ◄──── wire_all() ─────── get_output(local_port, REQ)

get_output(RSP)  ──── wire_all() ────────► set_input(local_port, RSP)
set_input(RSP)   ◄──── wire_all() ─────── get_output(local_port, RSP)
```

### 5.2 連接類型比較

| Connection Type | A 側 Interface | B 側 Interface | Wiring |
|----------------|---------------|---------------|--------|
| Router ↔ Router | `Router_Interface::get_output(port)` | `Router_Interface::set_input(port)` | 對稱交換 |
| NI ↔ Router | `NI_Interface::get_output(ch)` | `Router_Interface::set_input(local_port)` | 對稱交換 |

**兩者使用完全相同的 `PortOutput<Mode>` 信號定義和 wiring 機制，無需區分。**

### 5.3 Credit 初始化（Version B）

Credit mode 下，每端的 CreditCounter 初始值 = 對端 input buffer 的 depth per VC：

```cpp
// CppRouter 內部初始化 LOCAL port 的 output credits
// credit = NI 端 input queue depth / NUM_VC
local_port.output_credits.init(ni_input_queue_depth / NUM_VC);

// CppNI 內部初始化 output credits
// credit = Router LOCAL port input buffer depth / NUM_VC
nmu_output_credits.init(router_buffer_depth / NUM_VC);
```

初始化後，credit counter 保證 sender 不會 overflow receiver 的 buffer。

---

## 6. Req/Rsp 物理分離

獨立的 Req/Rsp physical network 消除 request-response circular dependency，為 protocol-level deadlock avoidance 的主要機制。

在 abstract interface 中，Req/Rsp 的分離透過 `Channel` enum 表達：

```cpp
// Router_Interface::get_output(port, Channel::REQ) → Request network 信號
// Router_Interface::get_output(port, Channel::RSP) → Response network 信號
```

CppRouter 內部可實作為兩個獨立 sub-router，或以 Channel 為 key 的 port map。每個 sub-router 有 NUM_MESH_PORTS + NUM_LOCAL_PORTS 個 port，各 port 獨立的 input/output 信號。

---

## 7. 8-Phase Cycle Processing

Mesh 每個 simulation cycle 執行 8 個 phase，嚴格順序處理。以下定義每個 phase 的功能、硬體行為、data dependency 與 RTL 映射。

### 7.1 Phase 定義

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Cycle N                                           │
├───────┬──────────────────────┬──────────────────┬───────────────────────────┤
│ Phase │ Function             │ HW Behavior      │ RTL Mapping               │
├───────┼──────────────────────┼──────────────────┼───────────────────────────┤
│   1   │ Sample inputs        │ FF capture       │ posedge clk: FF latch     │
│   2   │ Clear input signals  │ Wire reset       │ (model housekeeping)      │
│   3   │ Update ready         │ Combinational    │ buffer_full → out_ready   │
│   4   │ Route & forward      │ Combinational    │ RC + VA + SA + ST         │
│   5   │ Propagate wires      │ Wire propagation │ Combinational wires       │
│   6   │ Clear accepted       │ FF update        │ posedge clk: output FF    │
│   7   │ Credit release       │ FF + wire        │ Credit return register    │
│   8   │ NI processing        │ Combinational+FF │ NI pipeline stages        │
└───────┴──────────────────────┴──────────────────┴───────────────────────────┘
```

### 7.2 Phase 詳細說明

#### Phase 1: Sample Inputs（FF Capture）

```cpp
// 對應 RTL: posedge clk 時 input flip-flop latch
for (auto& router : routers) {
    router.sample_all_inputs();
    // 每個 port: if (in_valid && out_ready) → push in_flit to input_buffer
}
```

- **硬體行為：** Input buffer 的 write enable = `in_valid & out_ready`，在 clock edge 將 `in_flit` 寫入 buffer。
- **Data dependency：** 讀取 Cycle N-1 Phase 5 propagate 設定的 `in_valid`, `in_flit` 信號。
- **關鍵約束：** Sample 必須在 Phase 5 (Propagate) **之前**執行。Phase 1 取樣的是上一 cycle Phase 5 propagate 的結果。

#### Phase 2: Clear Input Signals（Model Housekeeping）

```cpp
for (auto& router : routers) {
    router.clear_all_input_signals();
    // in_valid = false; in_flit = {}; (防止同一 flit 被重複 sample)
}
```

- **硬體行為：** 無直接 RTL 對應。此為 C++ model 的 housekeeping — RTL 中 FF 自然只 latch 一次。
- **Data dependency：** Phase 1 完成後執行。

#### Phase 3: Update Ready（Combinational Logic）

```cpp
for (auto& router : routers) {
    router.update_all_ready();
    // 每個 port: out_ready = !input_buffer.full()
}
```

- **硬體行為：** Combinational logic — `out_ready` = NOT(buffer full flag)。
- **RTL 映射：** `assign out_ready = ~buffer_full;`（純 combinational）。
- **Data dependency：** 依賴 Phase 1 sample 後更新的 buffer 狀態。

#### Phase 4: Route & Forward（Combinational Pipeline：RC + VA + SA + ST）

```cpp
for (auto& router : routers) {
    router.route_and_forward(current_time);
    // 1. Route Computation (RC): 從 input_buffer peek flit, 計算 output port
    // 2. VC Allocation (VA): 分配目標 VC (Version B)
    // 3. Switch Allocation (SA): Wormhole arbiter 仲裁
    // 4. Switch Traversal (ST): 設定 out_valid, out_flit
    //    Version B: 檢查 output_credits.can_send(vc) 後 consume credit
}
```

- **硬體行為：** 多級 combinational logic（實際 RTL 可能拆為多個 pipeline stage）。
- **RTL 映射：** RC → VA → SA → ST 的 pipeline。本 C++ model 在單一 phase 內完成（behavioral model），等效 single-cycle router pipeline。
- **Data dependency：** 依賴 Phase 3 的 ready 狀態（判斷下游是否可接收）及 input buffer 內容。

#### Phase 5: Wire All（Signal Exchange via Abstract Interface）

```cpp
mesh.wire_all();
// 對每個 connection:
//   auto& out_a = a->get_output(port_a, ch);
//   auto& out_b = b->get_output(port_b, ch);
//   b->set_input(port_b, ch, out_a);  // A→B
//   a->set_input(port_a, ch, out_b);  // B→A
// Router-Router 和 NI-Router 使用相同機制
```

- **硬體行為：** Physical wire propagation — zero delay（combinational path）。
- **RTL 映射：** `assign` wire connections between modules。
- **Data dependency：** 依賴 Phase 4 設定的 `output.valid`, `output.flit` 與 Phase 3 設定的 `output.ready`（VR）。
- **結果將在下一 cycle Phase 1 被 sample。**

#### Phase 6: Clear Accepted Outputs（FF Update）

```cpp
for (auto& router : routers) {
    router.clear_accepted_outputs();
    // 每個 port: if (out_valid && in_ready) → clear out_valid, out_flit
    //            if (output_buffer not empty) → advance output_buffer
}
```

- **硬體行為：** Output register 在確認 handshake 完成後清除。
- **RTL 映射：** `posedge clk: if (out_valid & in_ready) out_valid <= 0;`
- **Data dependency：** 依賴 Phase 5 propagate 的 `in_ready`。

#### Phase 7: Credit Release（Version B）

```cpp
// Credit return 已包含在 PortOutputCredit.credit[vc] 中
// Phase 5 的 wire_all() 已將 credit 信號交換至對端
// 此 phase 處理 credit counter 更新
for (auto& router : routers) {
    router.process_credit_returns();
    // 每個 port: if (input.credit[vc]) → output_credits.release(vc)
    // 每個 port: if (buffer consumed flit from vc) → output.credit[vc] = true
}
```

- **硬體行為：** Credit return register update。
- **RTL 映射：** `posedge clk: credit_counter[vc] <= credit_counter[vc] + credit[vc];`
- **Data dependency：** 依賴 Phase 1 的 flit consumption（buffer pop）觸發 credit return。
- **Version A 中此 phase 為 no-op。**

#### Phase 8: NI Processing（NI Pipeline）

```cpp
for (auto& ni : network_interfaces) {
    ni.tick();
    // NMU: AXI req → flit pack → 更新 get_output(REQ) 的 valid/flit
    // NSU: 從 set_input(REQ) 讀取 flit → AXI master req → collect rsp → 更新 get_output(RSP)
}
// NI output 信號在下一 cycle Phase 5 由 wire_all() 傳遞至 Router
```

- **硬體行為：** NI 內部 pipeline（AXI ↔ flit conversion）。
- **Data dependency：** NI 讀取 Phase 5 wire_all() 設定的 input 信號，產生 NI output。
- **Injection latency：** NI 在 Phase 8 設定 output → 下一 cycle Phase 5 wire_all() 傳播至 Router → 再下一 cycle Phase 1 Router sample。因此 NI injection 相對 Router-Router 的 zero-cycle propagation 多出 **1 cycle 額外延遲**（NI output → Router input 需跨越 cycle boundary）。此行為與 Section 3.6 Latency Model 中的 zero-cycle propagation 一致：wire_all() 本身是 zero-delay，額外延遲來自 NI tick 在 Phase 8（cycle 尾端）產生 output。

### 7.2.1 Credit Return 精確時序

Credit-based flow control（Version B）的 credit return 時序必須精確定義，避免 implementation 歧義。

#### Credit 初始狀態

```
Initial credit[vc] = BUFFER_DEPTH_PER_VC    // e.g., BUFFER_DEPTH=4, NUM_VC=4 → 每 VC 1 entry
                                              // 或 BUFFER_DEPTH=4, NUM_VC=1 → 每 VC 4 entries
Credit counter range: [0, BUFFER_DEPTH_PER_VC]
Credit underflow (< 0): assert failure — 設計 bug，simulation 中止
```

#### Credit 消耗與歸還的 Phase 對應

| 事件 | Phase | 動作 | 說明 |
|------|-------|------|------|
| Flit 被 downstream 接受 | Phase 1 (Sample) | `input_buffer.push(flit)` | Downstream input buffer 佔用一個 slot |
| Input buffer slot 釋放 | Phase 4 (Route & Forward) | `input_buffer.pop()` | Flit 被 crossbar consume，buffer slot 空出 |
| Credit return 產生 | Phase 7 (Credit Release) | `credit_valid_out = true` | 通知上游 buffer slot 已釋放 |
| Credit return 傳播 | Phase 5 (Wire All) | `wire_all()` | 經 abstract interface 信號交換傳播至上游 |
| 上游收到 credit | Phase 7 (Credit Release) | `output_credits.release(vc)` | 上游 credit counter +1 |

#### 時序圖

```
Cycle N                                    Cycle N+1
────────────────────────────────           ─────────────
Phase 1: Flit F sampled into              Phase 1: 上游可使用
         downstream input buffer                    歸還的 credit
         (credit consumed at send time)             發送新 flit
    ↓
Phase 4: Flit F popped from
         input buffer (crossbar switch)
    ↓
Phase 7: Credit return generated
         → propagated via wire_all()
         → upstream credit[vc] += 1
                                           Phase 1: 上游 sees credit > 0
                                                    → can send new flit
```

**Credit return latency = 同 cycle 內歸還**。Flit 在 Phase 4 被 consume 後，Phase 7 即歸還 credit。上游在 **Cycle N+1 的 Phase 4** 即可使用此 credit 發送新 flit。

#### Credit 不變量 (Invariants)

```cpp
// 任意時刻，以下等式成立：
// upstream.credit[vc] + downstream.input_buffer.count(vc) + in_flight_flits(vc)
//   == BUFFER_DEPTH_PER_VC
//
// 其中 in_flight_flits = 正在 link 上傳輸（已發送但未被 sample）的 flit 數

// Credit 安全檢查
void assert_credit_invariant(int vc) const {
    assert(output_credits.credits[vc] >= 0);                    // 不可 underflow
    assert(output_credits.credits[vc] <= BUFFER_DEPTH_PER_VC);  // 不可超過初始值
}
```

#### Version A (Valid/Ready) 的等效行為

Version A 不使用 explicit credit。`ready` 信號在 Phase 3 (Update Ready) 根據 `!input_buffer.full()` 直接設定，行為等效於 combinational credit — 無 1-cycle latency。Phase 7 在 Version A 下為 no-op。

### 7.3 Data Dependency 圖

```
Cycle N-1                              Cycle N                              Cycle N+1
─────────                              ───────                              ─────────
Phase 5: wire_all()  ──────────────►  Phase 1: sample inputs
                                          │
                                          ▼
                                      Phase 2: clear input signals
                                          │
                                          ▼
                                      Phase 3: update ready/credit ◄── buffer state from Phase 1
                                          │
                                          ▼
                                      Phase 4: route & forward ◄── ready/credit from Phase 3
                                          │                         buffer from Phase 1
                                          ▼
                                      Phase 5: wire_all() ─────────► Phase 1 (Cycle N+1)
                                          │
                                          ▼
                                      Phase 6: clear accepted ◄── input from Phase 5
                                          │
                                          ▼
                                      Phase 7: credit release ◄── consumption from Phase 1
                                          │
                                          ▼
                                      Phase 8: NI tick() ◄── input from Phase 5
```

### 7.4 RTL posedge clk 映射

在 RTL 中，一個 clock cycle 的行為對應：

```
posedge clk:
  ┌─ (1) Input FF latch (Phase 1: sample)
  ├─ (2) Output FF update (Phase 6: clear accepted)
  └─ (3) Credit counter update (Phase 7: credit release)

Between posedge clk (combinational):
  ┌─ (4) out_ready = ~buffer_full (Phase 3)
  ├─ (5) RC → VA → SA → ST pipeline (Phase 4)
  └─ (6) Wire propagation (Phase 5)
```

C++ model 將 RTL 的並行行為拆解為 8 個循序 phase，以 phase ordering 保證因果關係正確。Phase 1 / 6 / 7 對應 `posedge clk` 的 sequential logic；Phase 3 / 4 / 5 對應 combinational logic。

### 7.5 完整 Cycle C++ 虛擬碼

```cpp
template<FlowControlMode Mode>
void Mesh<Mode>::process_cycle(int current_time) {
    // Phase 1-4: 所有 Router tick（內部執行 sample → clear → ready → route）
    for (auto& [coord, router] : routers)
        router->tick();

    // Phase 5: Wire all — 透過 abstract interface 交換信號
    wire_all();  // 見 Section 3.4

    // Phase 6: 所有 Router 處理 handshake completion
    for (auto& [coord, router] : routers)
        router->post_wire();  // clear accepted outputs based on input

    // Phase 7: Credit release (Version B only; no-op for Version A)
    for (auto& [coord, router] : routers)
        router->process_credit_returns();

    // Phase 8: 所有 NI tick（AXI ↔ flit conversion）
    for (auto& [coord, ni] : network_interfaces)
        ni->tick();
}
```

> **注意：** Phase 1-4 封裝在 `Router_Interface::tick()` 內部，Phase 6-7 為 post-wire 處理。Simulation Driver 只呼叫 abstract interface method，不直接操作內部信號。具體 phase 分割由 `CppRouter` 實作決定。

---

## 8. Channel\<T\>（帶延遲的 Link Model）

Channel 是一個帶有可配置延遲的 pipeline FIFO，取代原本 `wire_all()` 的 zero-cycle hard-wired 直連，模擬 Router ↔ Router 與 NI ↔ Router 之間的 link propagation delay。

### 8.1 Channel 介面

```cpp
template<FlowControlMode Mode>
class Channel {
public:
    explicit Channel(int latency = 1);

    // Sender side: 寫入 output data
    void Send(const PortOutput<Mode>& data);

    // Receiver side: delay cycles 後可用
    const PortOutput<Mode>& Receive() const;
    bool HasData() const;

    // Simulation: 每 cycle 呼叫
    void ReadInputs();     // Latch incoming（Phase 5 前半）
    void WriteOutputs();   // Shift pipeline, output oldest（Phase 5 後半）

private:
    int latency_;
    std::deque<PortOutput<Mode>> pipeline_;  // delay line
};
```

### 8.2 Latency 行為

| latency | 行為 | RTL 映射 | 用途 |
|---------|------|---------|------|
| 0 | Zero-cycle（combinational wire） | `assign` 直連 | 等同原本 wire_all() |
| 1 | 下一 cycle 可見（預設） | 1-stage pipeline register | 標準 hop-to-hop delay |
| N>1 | N cycle 後可見 | N-stage pipeline register | Long-wire / pipeline link |

### 8.3 Mesh 中的 Channel 使用

```cpp
template<FlowControlMode Mode>
class Mesh {
    // ...
    std::vector<Channel<Mode>> router_channels_;   // Router ↔ Router links
    std::vector<Channel<Mode>> ni_channels_;       // NI ↔ Router links
};
```

Mesh 的 `wire_all()` 改為透過 Channel 傳遞：

```cpp
template<FlowControlMode Mode>
void Mesh<Mode>::wire_all() {
    // Phase 5a: 讀取所有 component outputs → Channel.Send()
    for (auto& link : router_channels_) {
        auto& out = link.src->get_output(link.src_port, link.ch);
        link.channel.Send(out);
    }
    // Phase 5b: Channel pipeline shift → 輸出延遲後的 data
    for (auto& link : router_channels_) {
        link.channel.ReadInputs();
        link.channel.WriteOutputs();
        if (link.channel.HasData()) {
            link.dst->set_input(link.dst_port, link.ch, link.channel.Receive());
        }
    }
    // NI channels 同理
}
```

---

## 9. TrafficManager（中央協調器）

TrafficManager 作為 NocSystem 與 Mesh 之間的中介層，管理 transaction 的完整生命週期：拆解、注入、追蹤、收集。

### 9.1 架構定位

```
NocSystem API (Layer 2)
     ↓ submit_write / submit_read / load_traffic
TrafficManager
     ├── Transaction → Packet → Flit 拆解
     ├── Injection Queue (per NMU)
     ├── Outstanding Transaction Tracking
     ├── Completion Collection (from NMU RoB)
     ├── Golden Capture (write → GoldenManager)
     └── Statistics Aggregation
     ↓ inject flit to NMU
NMU (via Mesh)
```

### 9.2 TrafficManager 介面

```cpp
class TrafficManager {
public:
    TrafficManager(Mesh<Mode>& mesh, GoldenManager& golden,
                   MetricsCollector& metrics);

    // 接收 high-level transaction，拆為 packet/flit
    TxnHandle submit_write(uint64_t addr, const uint8_t* data,
                           size_t len, uint8_t axi_id);
    TxnHandle submit_read(uint64_t addr, size_t len, uint8_t axi_id);
    TxnHandle submit_multicast_write(uint64_t base_addr,
                                     const uint8_t* data, size_t len,
                                     const std::vector<uint8_t>& dst_nodes);

    // 從 JSON traffic file 載入
    void load_traffic(const std::string& json_path);

    // 每 cycle 調用：inject flits to NMU, collect completions from NMU
    void tick(int current_cycle);

    // Query
    bool is_complete(TxnHandle h) const;
    bool all_complete() const;
    TxnStatus get_status(TxnHandle h) const;
    const std::vector<uint8_t>& get_read_data(TxnHandle h) const;

private:
    struct Transaction {
        TxnHandle handle;
        TxnType type;           // WRITE / READ / MULTICAST
        uint64_t addr;
        std::vector<uint8_t> data;
        uint8_t axi_id;
        uint8_t src_node;
        TxnStatus status;       // PENDING / INJECTING / WAITING_RESPONSE / COMPLETE / ERROR
        int submit_cycle;
        int inject_cycle;       // 排程注入 cycle（絕對值或 "after:N"）
        int complete_cycle;
    };

    TxnHandle next_handle_ = 0;
    std::deque<Transaction> pending_txns_;                       // 待注入
    std::unordered_map<TxnHandle, Transaction> active_txns_;    // 進行中
    std::unordered_map<TxnHandle, Transaction> completed_txns_; // 已完成

    Mesh<Mode>&        mesh_;
    GoldenManager&     golden_;
    MetricsCollector&  metrics_;
};
```

### 9.3 Transaction Lifecycle

```
PENDING → INJECTING → WAITING_RESPONSE → COMPLETE / ERROR
   ↑         ↑              ↑                ↑
   │    flits 開始注入  所有 flits 已注入   收到 response
   │    到 NMU          等待 B/R          status 確定
   │
submit 時建立
```

---

## 10. Allocator 抽象（Pluggable）

定義 abstract Allocator base class，允許 runtime 切換不同的 allocation 策略。

### 10.1 Allocator Interface

```cpp
class Allocator {
public:
    virtual ~Allocator() = default;

    // 初始化 allocator（input_count × output_count 的配對）
    virtual void Init(int inputs, int outputs) = 0;

    // 加入一個 request（input i 請求 output o，priority p）
    virtual void AddRequest(int input, int output, int priority = 0) = 0;

    // 執行 allocation，回傳 grant 結果
    // grants[input] = output（-1 表示未分配）
    virtual void Allocate() = 0;

    // 查詢 grant 結果
    virtual int OutputAssignment(int input) const = 0;
    virtual int InputAssignment(int output) const = 0;

    // 清除上一輪的 request/grant
    virtual void Clear() = 0;

    // Factory
    static std::unique_ptr<Allocator> Create(const std::string& type,
                                              int inputs, int outputs);
};
```

### 10.2 可用實作

| 實作 | Config String | 說明 | 來源 |
|------|-------------|------|------|
| `RoundRobinAllocator` | `"round_robin"` | Simple round-robin，per-output RR pointer | — |
| `iSLIPAllocator` | `"islip"` | Iterative Separable LP allocator (~500 行) | — |
| `QoSAwareRRAllocator` | `"qos_aware_rr"` | QoS priority + Round-Robin（預設 SW allocator） | 自定 |

### 10.3 在 CppRouter 中的使用

```cpp
template<FlowControlMode Mode>
class CppRouter : public Router_Interface<Mode> {
    // ...
    std::unique_ptr<Allocator> vc_allocator_;    // from config.vc_allocator
    std::unique_ptr<Allocator> sw_allocator_;    // from config.sw_allocator
};
```

---

## 11. Buffer / BufferState 分離

將 buffer data storage 與 credit tracking 分離為兩個獨立元件，提升設計清晰度。

### 11.1 Buffer（Data Storage）

```cpp
class InputBuffer {
public:
    explicit InputBuffer(int depth, int num_vc = 1);

    // Per-VC operations
    bool Push(int vc, const Flit& flit);
    Flit Pop(int vc);
    const Flit& Peek(int vc) const;
    bool Empty(int vc) const;
    bool Full(int vc) const;
    int  Size(int vc) const;

private:
    int depth_per_vc_;
    std::vector<std::deque<Flit>> buffers_;  // buffers_[vc]
};
```

### 11.2 BufferState（Credit Tracking）

```cpp
class BufferState {
public:
    explicit BufferState(int depth, int num_vc = 1);

    // Credit management（sender side 呼叫）
    bool HasCredit(int vc) const;
    void ConsumeCredit(int vc);      // On flit send
    void ReturnCredit(int vc);       // On credit return pulse
    int  AvailableCredits(int vc) const;

    // VC state tracking
    void SetOccupied(int vc);        // VC 被 wormhole packet 佔用
    void SetFree(int vc);            // VC 釋放（TAIL 通過）
    bool IsFree(int vc) const;

private:
    int depth_per_vc_;
    std::vector<int>  credits_;      // credits_[vc]
    std::vector<bool> occupied_;     // occupied_[vc] — wormhole lock
};
```

### 11.3 分離的好處

| 面向 | Buffer | BufferState |
|------|--------|-------------|
| 位置 | Downstream（本地） | Upstream（遠端 tracking） |
| 職責 | 儲存 flit data | 追蹤 credit / VC 狀態 |
| 存取 | Push/Pop/Peek | ConsumeCredit/ReturnCredit |
| 獨立性 | 純 data container | 純 state tracking |

> 此設計使 credit tracking 邏輯與 buffer data 操作完全正交，簡化 debug 與測試。

---

## 12. 參數總表

以下參數與 [04_flit.md](04_flit.md) 一致：

| Parameter | Value | Source |
|-----------|-------|--------|
| `FLIT_WIDTH` | 408 bits | 04_flit.md Section 1.2 |
| `HEADER_WIDTH` | 56 bits | 04_flit.md Section 1.2 |
| `PAYLOAD_WIDTH` | 352 bits | 04_flit.md Section 1.2 |
| `PHYSICAL_LINK_WIDTH` | 410 bits | 04_flit.md Section 4.1 |
| `MAX_VC` | 8 | 04_flit.md Section 2.2.9 |
| `NODE_ID_WIDTH` | 8 bits | 04_flit.md Section 1.2 |
| `VC_ID_WIDTH` | 3 bits | 04_flit.md Section 2.1 (Version B) |

Router 相關參數：

| Parameter | Typical Value | Description |
|-----------|---------------|-------------|
| `BUFFER_DEPTH` | 4 | Input buffer depth (flits) per port |
| `OUTPUT_BUFFER_DEPTH` | 0 or 2 | Output buffer depth (0 = disabled) |
| `NUM_MESH_PORTS` | 4 | N, E, S, W |
| `NUM_LOCAL_PORTS` | 1 (max 4) | LOCAL ports per router |
| `NUM_VC` | 4 | Virtual channels per port (Version B) |
| `BUFFER_DEPTH_PER_VC` | `BUFFER_DEPTH / NUM_VC` | Buffer entries per VC |

---

## 相關文件

- [系統概述](01_overview.md)
- [Router 規格](02_router.md) — Port 定義、XY Routing、Wormhole Arbiter、CppRouter Pipeline Stages
- [Network Interface 規格](03_network_interface.md) — NI 功能模組、AXI Protocol Conversion、CppNI Functions
- [Flit Format](04_flit.md) — **基準文件**：Header/Payload 格式、Physical Link 定義、Flow Control 版本
- [模擬規格](08_simulation.md) — NocSystem API (6 組)、I/O Pattern、DPI-C Bridge、Replaceable Components
- [QoS Design](10_qos.md) — QoS arbitration 策略
- [架構圖](../images/noc_env.txt) — 系統架構圖
