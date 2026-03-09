---
document_id: NOC-SPEC-03
title: Router
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [02_flit.md]
---

# Router

## 1. Overview

### 1.1 用途

Router 是 NoC Mesh 的封包轉發單元，負責接收 flit、路由決策、仲裁、crossbar switching 與轉發。每個 Router 由兩個結構對稱的子 Router 組成 — **ReqRouter**（AW/W/AR）與 **RspRouter**（B/R），共用相同的 routing、arbitration、crossbar 邏輯。

### 1.2 在系統中的位置

```
           N (to y+1)
           |
   W ---- [Router] ---- E
  (x-1)   |    |       (x+1)
           S    L
         (y-1) (Local NI)
```

Router 透過 4 個 mesh direction port（N/S/E/W）連接相鄰 Router，透過 0~4 個 LOCAL port 連接 Network Interface (NI)。Mesh 邊界不存在的方向 port 不連接。

### 1.3 設計目標

- Cycle-accurate behavior model，可與 RTL co-simulation
- XY Routing 保證 deadlock-free
- Wormhole switching 保證封包完整性
- QoS-aware arbitration 支援 16 級優先
- Multicast 支援 independent per-port handshake
- In-Network Reduction（RspRouter only）合併 multicast B response
- Pluggable allocator 允許替換仲裁策略

---

## 2. Architecture & Block Diagram

### 2.1 Block Diagram

```
                     Physical Links (per direction)
                          N port
                     in ◄──┤├──► out
                           │
      W port          ┌────┴────┐          E port
 in ◄──┤├──► out ────►│         │◄──── in ◄──┤├──► out
                      │ Crossbar │
 out ◄──┤├──► in ◄────│  (N×N)  │────► out ◄──┤├──► in
                      │         │
                      └────┬────┘
                           │
                     out ◄──┤├──► in
                          S port
                           │
                     in ◄──┤├──► out
                        L0~L3 (LOCAL ports)

 Each port data path:
   Ingress → InputBuffer(depth=4) → Crossbar → OutputBuffer(depth=2*) → Egress
   * OUTPUT_BUFFER_DEPTH=0 時為 wire-through（無 output buffer）
```

> **Implementation Note：** Router 為 template class `Router<FlowControlMode>`，所有 sub-module 均透過 composition 組合（非繼承）。Crossbar 大小隨 `NUM_ROUTES` 動態決定。

### 2.2 Sub-Module 職責

| Sub-Module | 職責 |
|------------|------|
| InputBuffer | Per-port, per-VC flit 儲存。Push/Pop/Peek |
| BufferState | Per-output credit tracking（sender side），與 InputBuffer 分離 |
| RouteCompute | XY routing + Multicast RCR routing |
| VC Allocator | HEAD flit 的 VC 分配（Credit-Based mode only，pluggable） |
| SW Allocator | Per-output switch arbitration（QoS-aware RR，pluggable） |
| Crossbar | N×N non-blocking switch fabric |
| PathLock | Per-input wormhole lock FSM |
| MulticastTracker | Per-input multicast handshake tracking |
| ReductionSync | 同步等待 multicast B response（RspRouter only） |
| ReductionArbiter | 合併多個 B response 為一（RspRouter only） |

### 2.3 ReqRouter / RspRouter

每個邏輯 Router 包含兩個獨立子 Router，使用獨立 physical link，消除 request-response circular dependency：

```
Router A  ════ Req Link ════►  Router B
          ◄════ Req Link ════
          ════ Rsp Link ════►
          ◄════ Rsp Link ════
```

RspRouter 額外包含 ReductionSync + ReductionArbiter，處理 multicast B response 的 in-network reduction。

---

## 3. Parameters

Router 使用 compile-time parameter 定義結構，由上層 instantiation 時傳入。

### 3.1 Structural Parameters（RTL parameter 對應）

以下參數對應 RTL module 的 `parameter`，在 C++ model 中為 compile-time 常數或 NocConfig 配置值：

| Parameter | Type | Default | RTL 對應 | Description |
|-----------|------|---------|---------|-------------|
| `NUM_ROUTES` | int | 5 | `NUM_ROUTES` | Port 總數（含 mesh + local） |
| `NUM_INPUT` | int | NUM_ROUTES | `NUM_INPUT` | Input port 數（可與 output 不同） |
| `NUM_OUTPUT` | int | NUM_ROUTES | `NUM_OUTPUT` | Output port 數 |
| `NUM_VIRT_CHANNELS` | int | 1 | `NUM_VIRT_CHANNELS` | Virtual channel 數量 |
| `NUM_PHYS_CHANNELS` | int | 1 | `NUM_PHYS_CHANNELS` | Physical channel 數量（≤ NUM_VIRT_CHANNELS） |
| `INPUT_BUFFER_DEPTH` | int | 4 | `INPUT_BUFFER_DEPTH` | Input buffer 深度 (flits) |
| `OUTPUT_BUFFER_DEPTH` | int | 2 | `OUTPUT_BUFFER_DEPTH` | Output buffer 深度 (0 = bypass) |
| `ROUTE_ALGO` | enum | XYRouting | `ROUTE_ALGO` | Routing algorithm（見 3.2） |
| `ID_WIDTH` | int | 8 | `ID_WIDTH` | Node ID 寬度 (bits) |
| `XY_ROUTE_OPT` | bool | true | `XY_ROUTE_OPT` | 禁止 Y→X 連接（synthesis 優化） |
| `NO_LOOPBACK` | bool | true | `NO_LOOPBACK` | 禁止 loopback（input==output） |
| `VC_IMPL` | enum | VcNaive | `VC_IMPL` | VC 實作方式（見 3.3） |
| `EN_MULTICAST` | bool | false | `EN_MULTICAST` | 啟用 multicast 支援 |
| `EN_REDUCTION` | bool | false | `EN_REDUCTION` | 啟用 in-network reduction |

### 3.2 ROUTE_ALGO 列舉

| Value | Description |
|-------|-------------|
| `XYRouting` | XY 維度序路由（本設計預設） |
| `IdTable` | 基於 routing table 的 ID 路由 |
| `SourceRouting` | Source 端計算完整路徑（header 攜帶 route list） |

本設計使用 `XYRouting`。`IdTable` 與 `SourceRouting` 為擴展預留。

### 3.3 VC_IMPL 列舉

| Value | Description | 特性 |
|-------|-------------|------|
| `VcNaive` | Valid/Ready 直通 | 最簡單，可能產生跨模組 combinational path |
| `VcCredit` | Credit-based flow control | 切斷 in-to-out path，需 INPUT_BUFFER_DEPTH ≥ 3 |
| `VcPreemptValid` | Preemptive valid | 切斷 in-to-out path，INPUT_BUFFER_DEPTH ≥ 2 即可 |

### 3.4 Type Parameters

| Parameter | Description | 來源 |
|-----------|-------------|------|
| `flit_t` | Flit data type（408 bits） | [Flit Format](02_flit.md) |
| `id_t` | Node ID type（`logic[ID_WIDTH-1:0]`） | ID_WIDTH 推導 |
| `addr_rule_t` | Address routing rule（IdTable 用） | 上層定義 |
| `payload_t` | Flit payload type（352 bits） | [Flit Format](02_flit.md) |

### 3.5 Simulation Parameters（C++ model 專用）

以下參數僅存在於 C++ model，用於 pipeline delay 模擬，無直接 RTL 對應：

| Parameter | Source | Default | Description |
|-----------|--------|---------|-------------|
| `ROUTING_DELAY` | NocConfig | 0 | Route computation 額外 cycle |
| `VC_ALLOC_DELAY` | NocConfig | 0 | VC allocation 額外 cycle |
| `SW_ALLOC_DELAY` | NocConfig | 0 | Switch allocation 額外 cycle |
| `REDUCTION_TIMEOUT` | NocConfig | 1000 | ReductionSync 超時 cycle 數 |
| `CREDIT_TIMEOUT` | NocConfig | 10000 | Credit starvation 偵測 cycle 數（0 = 停用） |
| `MULTICAST_TIMEOUT` | NocConfig | 10000 | Multicast handshake 超時 cycle 數（0 = 停用） |

**參數約束：**
- `NUM_PHYS_CHANNELS ≤ NUM_VIRT_CHANNELS`
- `INPUT_BUFFER_DEPTH_PER_VC = INPUT_BUFFER_DEPTH / NUM_VIRT_CHANNELS`
- Crossbar 大小 = `NUM_INPUT × NUM_OUTPUT`
- `NUM_ROUTES = 0` 時 Router 不被 instantiate

---

## 4. Interface Specification

### 4.1 Port Direction 定義

Port 以固定 index 編碼（`route_direction_e`）：

| Index | Name | Direction | Connection | 備註 |
|-------|------|-----------|------------|------|
| 0 | North | y+1 | Router(x, y+1) | dst_y > cur_y |
| 1 | East | x+1 | Router(x+1, y) | dst_x > cur_x |
| 2 | South | y-1 | Router(x, y-1) | dst_y < cur_y |
| 3 | West | x-1 | Router(x-1, y) | dst_x < cur_x |
| 4 | Eject | local | NI (port_id=0) | dst == cur |
| 4+p | Eject+p | local | NI (port_id=p) | 額外 LOCAL port |

> `Eject+p` 設計允許多個 LOCAL port 自然擴展，無需特殊邏輯。

### 4.2 Router I/O Signals

Input 與 output 對稱，per-VC sideband + per-PhysCh data：

**Input Channels（from upstream）：**

| Signal | Dimension | Width | Description |
|--------|-----------|-------|-------------|
| `valid_i` | [NUM_INPUT][NUM_VC] | 1 bit each | Per-VC flit valid |
| `ready_o` | [NUM_INPUT][NUM_VC] | 1 bit each | Per-VC buffer ready |
| `data_i` | [NUM_INPUT][NUM_PHYS_CH] | FLIT_WIDTH each | Flit data（shared across VCs） |
| `credit_o` | [NUM_INPUT][NUM_VC] | 1 bit each | Per-VC credit return（VcCredit 時使用） |

**Output Channels（to downstream）：**

| Signal | Dimension | Width | Description |
|--------|-----------|-------|-------------|
| `valid_o` | [NUM_OUTPUT][NUM_VC] | 1 bit each | Per-VC flit valid |
| `ready_i` | [NUM_OUTPUT][NUM_VC] | 1 bit each | Per-VC downstream ready |
| `data_o` | [NUM_OUTPUT][NUM_PHYS_CH] | FLIT_WIDTH each | Flit data |
| `credit_i` | [NUM_OUTPUT][NUM_VC] | 1 bit each | Per-VC credit from downstream |

**Control Inputs：**

| Signal | Width | Description |
|--------|-------|-------------|
| `xy_id_i` | ID_WIDTH | 本 Router 的 XY 座標（XYRouting 使用） |
| `id_route_map_i` | NUM_ADDR_RULES × addr_rule_t | Routing table（IdTable 使用） |

### 4.3 Flow Control 行為

Flow control 行為由 `VC_IMPL` parameter 決定：

**VcNaive / VcPreemptValid（Valid/Ready）：**
- Handshake 成立：`valid_i[in][vc] && ready_o[in][vc]`
- `ready_o` 由 input buffer 的可用空間決定
- `credit_o` 恆為 1（不使用）

**VcCredit（Credit-Based）：**
- Sender 在 `credit > 0` 時發送，不依賴 `ready`
- `credit_o` 在 flit 被 input buffer pop 後的下一 cycle assert（1-cycle latency）
- Credit counter 初始值 = `INPUT_BUFFER_DEPTH / NUM_VIRT_CHANNELS`
- Invariant: `0 ≤ credit[vc] ≤ INPUT_BUFFER_DEPTH_PER_VC`

### 4.4 Physical Channel ↔ Virtual Channel 映射

| 條件 | 映射方式 |
|------|---------|
| `NUM_PHYS_CHANNELS == 1` | 所有 VC 共用一條 physical data wire |
| `NUM_PHYS_CHANNELS == NUM_VIRT_CHANNELS` | 每個 VC 有獨立 physical data wire |
| 其他 | 不支援（RTL 中為 `$fatal`） |

### 4.5 NI-Router Interface

NI-Router 連接使用**與 Router-Router 完全相同的 port interface**。NI 透過 Eject port 連接，由 Channel\<T\> 提供可配置的 link delay。

---

## 5. Functional Requirements

### FR-01: XY Routing

**描述：** 確定性路由演算法，先 X 後 Y，到達目標座標後轉 LOCAL。

**規則：**
1. `dst_x ≠ cur_x` → 沿 X 軸移動（EAST/WEST）
2. `dst_x == cur_x && dst_y ≠ cur_y` → 沿 Y 軸移動（NORTH/SOUTH）
3. `dst == cur` → LOCAL（由 `port_id` 選擇具體 LOCAL port）

**Y→X 轉向禁止：** 進入 Y 軸後禁止轉回 X 軸，形成 partial order，保證 deadlock-free。XY routing 的數學特性天然保證此條件，Y→X 檢查僅作 assertion。

**Invalid destination：** `dst_id` 超出 mesh 範圍時 drop flit + log warning。

> **Testability：** 以 4×4 mesh 測試所有 5 方向 output（N/S/E/W/LOCAL），驗證 edge node 不路由至不存在方向，注入 invalid `dst_id` 驗證 drop + warning。

### FR-02: Wormhole Switching

**描述：** HEAD flit 經仲裁後鎖定 input→output 路徑，BODY/TAIL 沿鎖定路徑直接轉發，TAIL 釋放路徑。

**Flit 類型判定：**

| Condition | Type | Behavior |
|-----------|------|----------|
| 無 lock 且 last=0 | HEAD | 需仲裁，獲勝後建立 path lock |
| 無 lock 且 last=1 | HEAD+TAIL | 仲裁後立即轉發，不鎖路 |
| 有 lock 且 last=0 | BODY | 沿鎖定路徑直接轉發 |
| 有 lock 且 last=1 | TAIL | 轉發後釋放 path lock |

**Path Lock（1-bit register `locked`）：**

```
                  valid & ready
                  & !last
┌────────────┐ ──────────────► ┌────────────┐
│  UNLOCKED  │                 │   LOCKED   │
│ (locked=0) │ ◄────────────── │ (locked=1) │
└────────────┘   valid & ready └──┬─────────┘
                 & last           │ valid & ready & !last
                                  └──┘ (stay LOCKED)
```

| 轉移 | 條件 | 說明 |
|------|------|------|
| UNLOCKED → LOCKED | `valid & ready & !last` | HEAD flit accepted，鎖定 output port |
| UNLOCKED → UNLOCKED | `valid & ready & last` | Single-flit packet，不鎖路 |
| LOCKED → LOCKED | `valid & ready & !last` | BODY flit，維持鎖定 |
| LOCKED → UNLOCKED | `valid & ready & last` | TAIL flit accepted，立即釋放 |

鎖定期間 `route_sel` 保持不變（使用 registered value），BODY/TAIL 不參與 routing 與 arbitration。

**AXI Channel 對應：**

| AXI Channel | Network | Flit Count | Wormhole |
|-------------|---------|-----------|----------|
| AW | REQ | 1 | 不鎖路（last=1） |
| W (awlen+1 beats) | REQ | awlen+1 | HEAD→BODY→TAIL |
| AR | REQ | 1 | 不鎖路 |
| B | RSP | 1 | 不鎖路 |
| R (arlen+1 beats) | RSP | arlen+1 | HEAD→BODY→TAIL |

AW 與 W 為獨立 wormhole packet。AW→W ordering 由 NMU 注入順序保證，Router 無需 AW/W 關聯邏輯。

> **Testability：** 驗證 PathLock FSM 所有 4 種轉移（UNLOCKED↔LOCKED），測試 single-flit（HEAD+TAIL）不鎖路，測試 multi-flit burst 鎖路與 TAIL 釋放。

### FR-03: Pipeline Stages

**描述：** Router 每 cycle 執行 8-phase pipeline（完整定義見 [Simulation Platform](08_simulation.md) §6）。Phase 1~4 封裝在 `tick()` 內，Phase 6~7 為 post-wire 處理。

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Sample | 取樣上一 cycle wire_all() 的 input → push to buffer |
| 2 | Clear Inputs | 清除 input 信號避免重複取樣 |
| 3 | Update Ready | 更新 ready/credit（ready = !buffer.full()） |
| 4 | Route & Forward | RC → VA → SA → ST → OQ sub-stages |
| 5 | Wire All | Channel\<T\> 交換所有元件 output → 對端 input |
| 6 | Clear Accepted | 確認 handshake，清除已接受的 output |
| 7 | Credit Update | 收到 credit return 後更新 counter（Credit-Based mode） |
| 8 | NI Process | NI 處理 AXI ↔ flit conversion |

> **Implementation Note：** `tick()` 封裝 Phase 1-4，`post_wire()` 封裝 Phase 6-7。Simulation driver 在所有元件 `tick()` 後呼叫 `wire_all()`，再呼叫所有元件 `post_wire()`。

**Phase 4 子步驟（per-flit state machine）：**

| Sub-stage | Function | Description |
|-----------|----------|-------------|
| 4a RC | Route Computation | XY routing，計算 output port |
| 4b VA | VC Allocation | VC 分配（Credit-Based mode，pluggable Allocator） |
| 4c SA | Switch Allocation | Switch arbitration（QoS-aware，pluggable） |
| 4d ST | Switch Traversal | Crossbar switching |
| 4e OQ | Output Queuing | 寫入 output buffer（或 wire-through） |

每個 head flit 在各 sub-stage 等待 `*_delay` cycles（NocConfig 參數）。Delay 期間 flit 佔 input buffer slot，同 VC 後續 flit 被 back-pressure。Body/tail flit 沿 path lock 直接進入 ST。所有 delay=0 時等效 single-cycle pipeline。

> **Testability：** 分別設定 `ROUTING_DELAY=0/1/2`，驗證 head flit latency 符合 `1 + sum(delays)` cycles。驗證 body/tail 不受 delay 影響。

### FR-04: Arbitration

**描述：** Per-output QoS-aware Round-Robin 仲裁。

**策略：**
1. Higher `qos` value wins（qos=15 highest）
2. Same qos：Round-Robin（per-output RR pointer）
3. Wormhole locked BODY/TAIL 不參與仲裁，直接通過

**QoS 與 Wormhole 交互：** 低 QoS 長 packet（最大 awlen=255, 256 flits）鎖路時，高 QoS HEAD 須等待直到該 packet 傳完（priority inversion）。這是 wormhole switching 的正常行為。緩解機制：age-based promotion（optional）、per-QoS VC（future）。

**Anti-starvation：** Round-Robin 保證同 QoS level 的 HEAD flit 不會被同 level 的其他 HEAD 無限期 skip。跨 QoS level 的 starvation 由 NI 端的 QoS Generator 緩解（Regulator mode 可自動提升長期低頻寬 master 的 QoS），詳見 [QoS](06_qos.md) §2.4 與 §6.1。

**Pluggable Allocator：** VC Allocator 與 SW Allocator 均使用抽象 `Allocator` interface，可透過 NocConfig 切換實作（`round_robin` / `islip` / `qos_aware_rr`）。Allocator 維持 1:1 interface，不感知 multicast。

> **Testability：** 注入 2 筆不同 QoS 的 HEAD flit 到同一 output port，驗證高 QoS 優先。注入同 QoS HEAD flit 多次，驗證 Round-Robin 公平性。

### FR-05: Buffer Management

**描述：** InputBuffer 與 BufferState 分離設計。

| 元件 | 位置 | 職責 |
|------|------|------|
| InputBuffer | Downstream（本地） | Per-port, per-VC flit data 儲存（Push/Pop/Peek） |
| BufferState | Upstream（遠端 tracking） | Per-VC credit counter + VC 佔用狀態追蹤 |

分離使 credit tracking 與 buffer data 操作完全正交。

**Output Buffer：** 位於 crossbar 與 physical link 之間。`OUTPUT_BUFFER_DEPTH ≥ 1` 時啟用，`= 0` 時為 wire-through（crossbar 直接驅動 output，flit 保留在 input buffer 直到 downstream 接受）。

> **Testability：** 測試 `INPUT_BUFFER_DEPTH` 邊界（push when full → assert），測試 `OUTPUT_BUFFER_DEPTH=0` 與 `=2` 兩種模式的行為差異。

### FR-06: Backpressure

**描述：** Backpressure 沿 wormhole path 反向逐 hop 傳播。

**Stall 觸發條件：**

| Condition | Effect |
|-----------|--------|
| Output buffer full | Crossbar 無法 switch 到此 output |
| Downstream not ready | Output buffer 無法 drain |
| Output port locked | 其他 input 的 wormhole lock 佔用此 output |

**HoL Blocking：** 被 stall 的 wormhole path 阻塞同 input buffer 的其他 HEAD flit。VC 可緩解（不同 VC 獨立 buffer）。

> **Testability：** 注入 burst 至 output buffer full，驗證 upstream stall 傳播。測試 stall 解除後 resume 正常傳輸。

### FR-07: Credit Management (Credit-Based Mode)

**描述：** Credit return 信號在 Phase 4 由 receiver 產生（flit pop 觸發），Phase 5 wire_all() 傳播至 sender，Phase 7 sender 更新 counter。Credit latency = 1 cycle，與 RTL posedge clk 行為一致。

**Credit counter update 時機：**
- Phase 4: Receiver pop flit → assert `credit[vc]`（combinational）
- Phase 5: wire_all() 將 credit signal 傳播至 upstream
- Phase 7: Upstream 收到 credit → `credit_counter[vc]++`

**Credit Starvation Detection：** 當某 VC 的 `credit_counter == 0` 持續超過 `CREDIT_TIMEOUT` cycles（預設 10,000）且該 VC 有待送 flit 時，視為異常。觸發時回報 error 至 NI CSR `ERR_STATUS`，不做自動 recovery — 正常壅塞（如長 wormhole packet 佔住 downstream buffer）不會觸發此 timeout。

> **Testability：** 驗證 credit 不變量 `credit + buffer_count + in_flight == DEPTH`。測試 credit=0 時 sender 停止發送。設定 `CREDIT_TIMEOUT` 驗證 starvation detection 觸發。

### FR-08: Multicast (Independent Per-Port Handshake)

**描述：** Multicast flit 的多 output 轉發採用 independent per-port handshake — 各 target output 可在不同 cycle 獨立完成，已完成的 output 立即釋放，input buffer 在**所有** target 完成後才 pop。

**MulticastTracker（per input port, bitmap register）：**

| Field | 寬度 | 說明 |
|-------|------|------|
| `expected` | NUM_OUTPUT bits | Routing function 決定的 target output set |
| `past_handshakes` | NUM_OUTPUT bits | 已完成 handshake 的 output set（registered） |
| `current_handshakes` | NUM_OUTPUT bits | 本 cycle 完成的 output set（combinational） |

**行為規則：**

```
current_handshakes = masked_valid & masked_ready
all_handshakes     = past_handshakes | current_handshakes
all_done           = (all_handshakes & expected) == expected
```

| 條件 | `past_handshakes` 更新 |
|------|----------------------|
| `all_done`（所有 target 完成） | 清零 → `0`，input buffer pop |
| 部分 target 完成 | 累積 → `past_handshakes | current_handshakes` |
| 無 handshake | 保持不變 |

**Valid 控制：** 對已完成 handshake 的 output 壓掉 valid（`valid & ~past_handshakes`），釋放該 output 給其他 traffic。

**Unicast 退化：** `expected` 僅 1 bit 為 1 時，行為與標準 single-output handshake 完全一致（`past_handshakes` 永遠為 0，single-cycle 完成）。

**時序範例（target = {N, E, L}）：**

```
Cycle 1: E accept ✓, N busy, L busy  → past = {E}
Cycle 2: N accept ✓, L busy          → past = {E, N}
Cycle 3: L accept ✓                  → all_done → past = 0, pop
```

| 特性 | 說明 |
|------|------|
| Output 早期釋放 | 已完成的 output 立即可服務其他 traffic |
| 無 collateral blocking | 不阻擋無關 input port |
| Arbiter 無感知 | Output arbiter 為標準 wormhole arbiter |
| Input HoL blocking | Flit 在 input buffer 停留至所有 output 完成 |

**Multicast Handshake Timeout：** 當 multicast flit 的 `past_handshakes != expected` 持續超過 `MULTICAST_TIMEOUT` cycles（預設 10,000）時，視為異常。觸發時回報 error 至 NI CSR `ERR_STATUS`，不做自動 recovery。

> **Testability：** 注入 multicast flit（target = 3 ports），驗證 3-cycle progressive completion。驗證 unicast 退化（expected 1-bit → single-cycle 完成）。測試 `MULTICAST_TIMEOUT` 觸發。

### FR-09: In-Network Reduction (RspRouter Only)

**描述：** Multicast write 的 B response 在 RspRouter 逐 hop 合併，Source NMU 僅收到 1 個已合併的 B response。

**流程：**

```
Forward:  NMU → ReqRouter → fan-out → NSU₀...NSUₙ₋₁
Backward: NSU₀...NSUₙ₋₁ → RspRouter → fan-in → NMU (1 merged B)
```

**Backward in_route_mask** = forward route_sel。Forward multicast fan-out 到的每個方向，backward 都會有 response 回來。

**ReductionSync：** 同步等待所有 expected inputs 到齊後觸發合併。支援 timeout 安全網（`REDUCTION_TIMEOUT` cycles，預設 1000），超時時以部分 response 合併，final bresp = SLVERR + warning log。

**ReductionArbiter 合併規則：**
- `bresp`：任一 SLVERR → 最終 SLVERR；全部 OKAY → OKAY
- `ecc_fail`：任一 true → 最終 true

**合併後 routing：** Unicast XY routing 向 `src_id` 方向輸出，保持 `commtype = ParallelReduction`，使下一 hop 繼續 reduction。

**Deadlock freedom 論證（需形式化驗證）：**
1. B response 是 single-flit（無 wormhole blocking）
2. 合併後走 unicast XY routing（deadlock-free 已證明）
3. Reduction tree 是 DAG（同一 transaction 無環）
4. Req/Rsp 物理分離

**潛在風險：** 跨 transaction B response 共享 buffer 可能形成 cross-transaction circular wait。Timeout 作為 runtime safety net。

> **Testability：** 注入 multicast write 至 3 個 NSU，驗證 NMU 收到 1 個 merged B response。測試部分 SLVERR → merged = SLVERR。測試 `REDUCTION_TIMEOUT` 觸發（模擬 NSU 不回應）。

### FR-10: Multi-Port LOCAL

**描述：** 每個 Router 可配置 0~4 個 LOCAL port，由 header `port_id`（2 bits）識別。

**行為：**
- XY routing 決定方向（N/S/E/W/LOCAL），不涉及 port_id
- 當方向 = LOCAL 時，`port_id` 選擇具體 LOCAL port
- 每個 LOCAL port 為完整獨立的 port 實例（buffer、credit、interface）
- Crossbar 大小隨 LOCAL port 數量擴展

**No-UTurn 擴展：**
- Mesh port 間：不允許同方向返回
- LOCAL port 間：不允許同 port_id self-loop，允許 inter-port 直連（L0→L1）
- Mesh↔LOCAL：不受 no-UTurn 限制

> **Testability：** 配置 2 個 LOCAL port，驗證 `port_id` 選擇正確 LOCAL port。驗證 L0→L1 inter-port 直連允許、L0→L0 self-loop 禁止。

### FR-11: Error Handling

**描述：** 模擬目標為快速偵測設計 bug（assert 中止），非 runtime recovery。

| Error | 偵測 | 處理 | 嚴重度 |
|-------|------|------|--------|
| Credit underflow | `credit[vc] < 0` | assert 中止 | FATAL |
| Buffer overflow | push when full | assert 中止 | FATAL |
| Invalid destination | dst_id 超出 mesh | Drop + warning | ERROR |
| Y→X turn violation | routing assertion | assert（不應發生） | FATAL |
| ECC failure | 偵測於 destination NI | Router 透通，不檢查 ECC | — |

Router 為 flit 透通轉發器，不檢查/修改 ECC。XY routing 保證 deadlock-free，不實作 deadlock detection/recovery。

> **Testability：** 注入 `credit < 0` 驗證 assert 觸發。注入 invalid `dst_id` 驗證 drop + warning log。

---

## 6. Performance

### 6.1 Latency

| Scenario | Best Case | Worst Case |
|----------|-----------|------------|
| Single flit, 1 hop, no contention | 1 + pipeline_delay cycles | — |
| Single flit, H hops, no contention | H + pipeline_delay cycles | — |
| Burst (N flits), H hops, no contention | H + N - 1 + pipeline_delay cycles | — |
| With contention | 取決於仲裁等待 | Bounded by burst length |

`pipeline_delay` = `ROUTING_DELAY + VC_ALLOC_DELAY + SW_ALLOC_DELAY`（NocConfig）

### 6.2 Throughput

| Metric | Target |
|--------|--------|
| Per-port, no contention | 1 flit/cycle |
| Sustained (uniform random) | >0.5 flits/cycle/node |

### 6.3 Blocking Analysis

| Blocking Source | Impact | 緩解 |
|----------------|--------|------|
| Wormhole path lock | 佔用 output port 直到 TAIL | Burst length 限制 |
| HoL blocking | 阻塞同 input buffer 的其他 HEAD | VC 分離 |
| Multicast HoL | Input blocked 直到所有 target 完成 | Output 早期釋放 |

---

## 7. Related Documents

- [System Overview](01_overview.md) — Mesh 拓撲與系統架構
- [Flit Format](02_flit.md) — Header/Payload 格式、physical link 定義
- [Network Interface](04_network_interface.md) — NI 與 LOCAL port 連接
- [Physical Channel](05_physical_channel.md) — 2-channel / 3-channel 架構
- [QoS Design](06_qos.md) — QoS arbitration policy
- [Multicast](07_multicast.md) — RCR routing algorithm
- [Simulation](08_simulation.md) — 8-phase cycle model、NocConfig、Channel\<T\>

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
