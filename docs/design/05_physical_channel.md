---
document_id: NOC-SPEC-05
title: Physical Channel Architecture
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [02_flit.md]
---

# Physical Channel Architecture

本文件描述 NoC 的 Physical Channel 架構，重點在 2-channel 與 3-channel 的設計差異。Flit 格式、header 欄位、payload layout 等定義請參見 [Flit Format](02_flit.md)。

---

## 1. 2-Channel Architecture（V1 預設）

本節描述 V1 預設的 Req/Rsp 雙通道架構，涵蓋 deadlock avoidance、對稱設計、頻寬分析與 flow control timing。

### 1.1 雙通道設計

NoC 採用 Request / Response 雙通道分離架構。每個邏輯 Router 內部由兩個結構對稱的獨立 Req Router / Rsp Router 組成：

| Router | Network | AXI Channels | Direction (typical) |
|------------|---------|--------------|---------------------|
| **Req Router** | Request | AW, W, AR | Master → Slave |
| **Rsp Router** | Response | B, R | Slave → Master |

兩個 Router（Req Router / Rsp Router）共用相同的 routing algorithm（XY routing）、arbitration policy（QoS-Aware Round-Robin）、crossbar structure，僅處理的 flit 類型不同（依 header `axi_ch` 欄位區分）。

### 1.2 架構圖

```
  Router A                                                Router B
  ┌──────────────────┐                                   ┌──────────────────┐
  │  ┌─────────────┐ │   Req Link (402b) ──────────►    │ ┌─────────────┐  │
  │  │  Req Router  │ │   ◄────────── Req Link (402b)    │ │  Req Router  │  │
  │  │  (400b flit)│ │                                   │ │  (400b flit)│  │
  │  └─────────────┘ │                                   │ └─────────────┘  │
  │  ┌─────────────┐ │   Rsp Link (402b) ──────────►    │ ┌─────────────┐  │
  │  │  Rsp Router  │ │   ◄────────── Rsp Link (402b)    │ │  Rsp Router  │  │
  │  │  (400b flit)│ │                                   │ │  (400b flit)│  │
  │  └─────────────┘ │                                   │ └─────────────┘  │
  └──────────────────┘                                   └──────────────────┘
        Total per router pair: 4 × 402 = 1,608 bits (bidirectional)
```

### 1.3 Deadlock Avoidance

Request 與 Response 使用**獨立 physical link**，從物理層面消除 protocol-level circular dependency：

```
Deadlock scenario (shared network):
  Node A ──[Req]──► buffer full ──► Node B
  Node A ◄──[Rsp]── buffer full ◄── Node B
                 ↑ circular wait ↓

Deadlock-free (separate networks):
  Request Network:   A ══[Req]══► B    (independent)
  Response Network:  A ◄══[Rsp]══ B    (independent)
```

Request buffer full 不影響 Response 傳輸，反之亦然。無需 software intervention 或 timeout recovery。

### 1.4 對稱設計優勢

Req/Rsp link 寬度完全對稱（均為 402 bits），兩個 Router（Req Router / Rsp Router）可複用同一硬體模組：

| Aspect | Benefit |
|--------|---------|
| RTL 模組 | 一套 XYRouter，實例化 2 次 |
| Verification | 一套 testbench 覆蓋兩個 network |
| Timing closure | 相同 critical path |
| Buffer sizing | 相同 SRAM macro |

### 1.5 Full-Duplex 頻寬

| Metric | Value |
|--------|-------|
| Per-link payload bandwidth | 256 bits = 32 bytes/cycle |
| Per-direction (Req + Rsp) | 2 × 400 = 800 bits/cycle |
| Bidirectional total | 4 × 400 = 1,600 bits/cycle |

### 1.6 Flow Control Timing

#### Valid/Ready Handshake

傳輸條件：`valid && ready` 同時為 high 時，flit 在該 cycle 被接受。

```
          Cycle 1    Cycle 2    Cycle 3    Cycle 4    Cycle 5
clk     ──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──
          └──┘  └──┘  └──┘  └──┘  └──┘

valid   ──────────────────────────────────┐
                                          └────────
flit    ══[F0]══[F1]══[F1]══[F1]══[F2]═════════════

ready   ──────────┐           ┌────────────────────
                  └───────────┘
                  ▲ backpressure (buffer full)

accept        ✓     ✗     ✗     ✓     ✓
              F0                F1    F2
```

- Cycle 1: valid=1, ready=1 → F0 accepted
- Cycle 2-3: ready=0 (downstream buffer full) → sender 保持 valid=1 且 flit=F1 不變
- Cycle 4: ready=1 → F1 accepted（stall 期間 flit 必須穩定，與 AXI 協議一致）
- Cycle 5: valid=1, ready=1 → F2 accepted

#### Credit-Based Flow Control

Sender 持有 credit counter，每送出一個 flit 消耗 1 credit。Downstream 消化 flit 後歸還 credit。

```
          Cycle 1    Cycle 2    Cycle 3    Cycle 4    Cycle 5
clk     ──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──
          └──┘  └──┘  └──┘  └──┘  └──┘

valid   ──────────────────┐
                          └──────────────────────
flit    ══[F0]══[F1]══[F2]═══════════════════════

credit_ret  __________________________┌──┐__________
(downstream → upstream)               │  ▲ F0 consumed, credit +1

credit_cnt   3     2     1     0     1
(upstream)               ▲ exhausted → stop sending
```

- credit_cnt=0 時 sender 停止發送，即使有待送 flit
- credit_ret 為 active-high pulse：平時 low，downstream 歸還 credit 時 pulse high 1 cycle
- Credit return latency = 1 cycle（Phase 4 pop → Phase 5 propagate → Phase 7 counter +1）
- 不變量：`credit + buffer_occupancy + in_flight == INPUT_BUFFER_DEPTH`

#### 8-Phase Cycle Timing

一個 cycle 內的 phase 執行順序（詳見 [Simulation Platform](08_simulation.md) §6）：

```
          ┌─── posedge clk ────────────────────── combinational ─────────────── posedge clk ───┐
          │                                                                                     │
Phase:    1          2          3          4              5           6          7          8
          Sample     Clear      Ready      Route &       Wire        Clear      Credit    NI
          Input      Inputs     Update     Forward       All         Accepted   Update    Process
          │                     │          │              │                      │
          ▼                     ▼          ▼              ▼                      ▼
     in_valid &&           out_ready   RC→VA→SA→ST    Channel<T>          credit_cnt++
     out_ready             = !full     pipeline       propagate           (Credit-Based)
     → push                           + credit_ret   flit/credit
                                       generate       exchange
```

---

## 2. Head-of-Line (HoL) Blocking 分析

HoL Blocking 是 wormhole switching 的主要效能瓶頸。本節分析 2-channel 架構下的 blocking 場景與緩解機制。

### 2.1 2-Channel 的 HoL Blocking

Request network 承載三種 AXI channel（AW, W, AR），共用同一 physical link 與 input buffer：

| 場景 | 原因 | 影響 |
|------|------|------|
| W burst 阻塞 AW/AR | Wormhole path lock 佔用 input→output 路徑 | AR 等待 W tail 傳完才能仲裁 |
| AW/AR 互相阻塞 | 競爭同一 output port | QoS-Aware RR 仲裁，落選者延遲 1 cycle |

W burst blocking duration 與 `awlen` 成正比：

| awlen | W Flits | Worst-case blocking cycles |
|-------|---------|---------------------------|
| 0 | 1 | 1 |
| 7 | 8 | 8 |
| 15 | 16 | 16 |

Response network（B + R）同理：R burst 會鎖定路徑阻塞 B flit，但 B 通常不具時間敏感性。

### 2.2 HoL Blocking 緩解機制

| 機制 | 說明 | 參考 |
|------|------|------|
| Virtual Channels (VC) | 不同 VC 擁有獨立 buffer，W/R 佔用不阻塞其他 VC | [Router](03_router.md) §3.3 |
| QoS priority | 高 qos HEAD flit 優先獲得路徑 | [QoS](06_qos.md) §5 |
| 3-Channel 架構 | 分離 narrow/wide 流量 | 本文件 Section 3 |

---

## 3. 3-Channel Architecture（V2 進階）

V2 架構引入第三條 physical channel，將 narrow 控制封包與 wide 資料封包分離，消除 wide-blocks-narrow HoL blocking 並支援 dual AXI 寬度介面（64b + 256b）。

### 3.1 設計動機

2-channel 使用統一 400-bit flit，對 address-only channel（AW/AR/B）浪費大量 padding。更重要的是，wide data burst（256b W/R）會阻塞 narrow control flit（AW/AR/B）。

3-channel 引入第三條 physical channel，將 **narrow 控制封包** 與 **wide 資料封包** 分離。

### 3.2 三通道定義

| Channel | Flit Width | Link Width | 承載 AXI Channels |
|---------|-----------|------------|-------------------|
| **req** (Narrow) | 164b | 166b | NarrowAW, NarrowW, NarrowAR, WideAR |
| **rsp** (Narrow) | 164b | 166b | NarrowR, NarrowB, WideB |
| **wide** | 400b | 402b | WideAW, WideW, WideR |

### 3.3 Dual AXI Interface

3-channel 模式支援兩種 AXI 寬度介面同時存在：

| Interface | Data Width | 典型用途 |
|-----------|-----------|---------|
| Narrow AXI | 64 bits | CPU cache line fetch、CSR access |
| Wide AXI | 256 bits | DMA bulk transfer、GPU texture |

### 3.4 Channel Mapping

| AXI Channel | → req | → rsp | → wide | 理由 |
|-------------|:-----:|:-----:|:------:|------|
| NarrowAW | ✓ | | | 窄地址請求 |
| NarrowAR | ✓ | | | 窄地址請求 |
| NarrowW | ✓ | | | 窄寫資料（64b） |
| WideAR | ✓ | | | 僅地址，小封包走 req |
| NarrowR | | ✓ | | 窄讀回應 |
| NarrowB | | ✓ | | 窄寫回應 |
| WideB | | ✓ | | 僅回應，小封包走 rsp |
| WideAW | | | ✓ | 與 WideW 同 channel 保序 |
| WideW | | | ✓ | 256b 大資料 |
| WideR | | | ✓ | 256b 大資料 |

**關鍵設計原則：**
- **WideAW + WideW 必須同 channel（wide）**：保證 write address/data ordering，避免 cross-channel reordering
- **WideAR 走 req（非 wide）**：AR 為 address-only 小封包，不佔 wide 頻寬
- **WideB 走 rsp（非 wide）**：B 為 response-only 小封包，不佔 wide 頻寬

### 3.5 Narrow Flit Format（164 bits）

Narrow flit 使用與 [Flit Format](02_flit.md) **相同的 48-bit header**，payload 縮小為 116 bits：

```
  163            116 115              0
  ┌─────────────────┬─────────────────┐
  │  Header (48b)   │  Payload (116b) │
  └─────────────────┴─────────────────┘
```

116-bit payload 可容納 address channel（AW/AR 約 108b）與 narrow data（64b data + 8b strb + 8b ECC + metadata）。Wide flit 格式與 2-channel 的 400-bit flit 完全相同。

### 3.6 axi_ch 欄位處理

3-channel 模式下 `axi_ch`（3 bits）不足以區分所有 10 種 channel。建議方案：

| axi_ch | 2-Channel | 3-Channel |
|--------|-----------|-----------|
| 0 | AW | NarrowAW / WideAW（由所在 channel 隱式區分） |
| 1 | W | NarrowW / WideW（由所在 channel 隱式區分） |
| 2 | AR | NarrowAR / WideAR（由所在 channel 隱式區分） |
| 3 | B | NarrowB / WideB（由所在 channel 隱式區分） |
| 4 | R | NarrowR / WideR（由所在 channel 隱式區分） |

接收端由 flit 所在的 physical channel（req/rsp vs wide）自動判斷 Narrow/Wide 來源，不需擴展 `axi_ch` 位寬。

### 3.7 Router 結構

3-channel 模式下，每個邏輯 Router 包含三個獨立 Router（Req Router / Rsp Router / Wide Router）：

| Router | Flit Width | 承載 AXI Channels |
|------------|-----------|-------------------|
| **Req Router** | 164b | NarrowAW, NarrowW, NarrowAR, WideAR |
| **Rsp Router** | 164b | NarrowR, NarrowB, WideB |
| **Wide Router** | 400b | WideAW, WideW, WideR |

Req Router 和 Rsp Router 共用同一 164-bit 寬度設計，可複用 RTL 模組。Wide Router 使用現有 400-bit 寬度設計。所有 Router 共用相同 routing algorithm 與 arbitration policy。

---

## 4. 2-Channel vs 3-Channel 比較

本節從 wire count、HoL blocking、RTL 複用性等維度量化比較兩種架構，提供選型依據。

### 4.1 Wire Count

| 設計 | Links per direction | Wires per direction | Per router pair |
|------|---------------------|---------------------|-----------------|
| 2-Channel | 2 × 402 | 804 | 4 × 402 = **1,608** |
| 3-Channel | 2 × 166 + 402 | 734 | 2 × 734 = **1,468 (-8.7%)** |

3-channel 反而比 2-channel **少 8.7% wire**，因為 narrow channels 使用適當寬度的 flit。

### 4.2 HoL Blocking 改善

| 場景 | 2-Ch | 3-Ch |
|------|:----:|:----:|
| NarrowW burst 阻塞 NarrowAW/AR | 有 | 有（仍共用 req） |
| WideW burst 阻塞 NarrowAW/AR | **有** | **無**（WideW 在 wide） |
| WideR burst 阻塞 NarrowB | **有** | **無**（WideR 在 wide） |

**主要改善：寬資料（256b）的 burst 不再阻塞窄控制封包（64b）。**

### 4.3 Trade-off 總結

| Aspect | 2-Channel | 3-Channel |
|--------|-----------|-----------|
| Req/Rsp/Wide Routers per logical router | 2 | 3 |
| Flit widths | 400b (uniform) | 164b + 400b |
| Wire count (per pair) | 1,608 | 1,468 (-8.7%) |
| RTL module reuse | Req Router = Rsp Router | Req Router = Rsp Router (164b) + Wide Router (400b) |
| HoL blocking (wide vs narrow) | 有 | 消除 |
| AXI interface support | Single width (256b) | Dual width (64b + 256b) |
| Buffer total (per router) | 2 × 5 ports × depth | 3 × 5 ports × depth |
| Complexity | 低 | 中 |

### 4.4 選擇建議

| 條件 | 推薦 | 理由 |
|------|------|------|
| 僅 256b AXI 介面 | 2-Channel | 無 narrow 流量，3-Ch 無效益 |
| 同時有 64b + 256b AXI | 3-Channel | 分離 narrow/wide，降低 HoL blocking |
| Wire budget 受限 | 3-Channel | 少 8.7% wires |
| Router 設計簡化 | 2-Channel | 2 Routers vs 3 Routers |
| FPGA prototype 優先 | 2-Channel | 較簡單，FPGA 資源較少 |

### 4.5 實作策略

- **V1**：採用 2-channel，確保架構簡潔可驗證
- **V2**：引入 3-channel，支援 Narrow+Wide 雙 AXI 介面

兩種模式由 compile-time configuration 選擇，core routing/arbitration 邏輯完全共用。

---

## Related Documents

- [Flit Format](02_flit.md) — Flit 結構、header 欄位定義、payload format、physical link format
- [Router Specification](03_router.md) — Router pipeline、Req Router/Rsp Router 架構、wormhole switching
- [Network Interface Specification](04_network_interface.md) — NI 的 AXI-to-Flit 轉換與 network 分流
- [Simulation Platform](08_simulation.md) — Abstract Interface、8-phase cycle model、Channel\<T\>
- [QoS Design](06_qos.md) — QoS-Aware arbitration policy

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
