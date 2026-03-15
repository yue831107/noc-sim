---
document_id: NOC-SPEC-06
title: QoS and Performance Monitoring
version: 2.0
status: Draft
last_updated: 2026-03-15
prerequisite: [02_flit.md, 03_router.md, 04_network_interface.md]
---

# QoS and Performance Monitoring

NoC 的 Quality of Service (QoS) 機制與效能監控設計。

---

## 1. Overview

QoS 機制確保不同 traffic 類型獲得適當的頻寬和延遲控制。系統透過 NI 端的 QoS Generator 產生 urgency level，由 Router 端的 Arbitration Scheme 進行仲裁，並由 Probe 單元收集效能資訊。

### 1.1 QoS 相關參數

#### 來自 Flit Format 的參數

| Parameter | Value | Source |
|-----------|-------|--------|
| `NODE_ID_WIDTH` | 8 (Fixed) | 02_flit.md |
| `QOS_WIDTH` | 4 (Fixed) | 02_flit.md |

#### QoS 模組專用參數

| Parameter | Range | Default | Description |
|-----------|-------|---------|-------------|
| `BW_COUNTER_WIDTH` | 16 ~ 32 | 24 | Bandwidth counter 寬度 |
| `URGENCY_WIDTH` | 2 ~ 4 | 3 | Urgency level 寬度 |
| `ERR_COUNTER_WIDTH` | 8 ~ 32 | 16 | Error counter 寬度 |

### 1.2 QoS 架構層級

| Layer | Component | 功能 |
|-------|-----------|------|
| Flit | Header `qos` (4 bits) | 攜帶 urgency level，Router arbitration 使用 |
| NI | QoS Generator | 產生 urgency level，寫入 flit header |
| NI | Probe Unit | 統計頻寬、延遲、錯誤 |
| Router | Arbitration Scheme | 根據 urgency 與策略進行仲裁 |
| System | CSR Interface | 軟體配置與監控 |

### 1.3 與 Flit Format 關聯

Header 中的 `qos` 欄位（4 bits, 16 levels）處理方式：

**Request Path (NMU)：**
```
AXI Master (awqos/arqos) → NMU (QoS Generator) → NoC Flit (qos in header)
```
- `QOS_MODE = Bypass`：直接使用 AXI awqos/arqos（預設行為）
- `QOS_MODE = Fixed/Limiter/Regulator`：由 QoS Generator 產生或調整

**Response Path (NSU)：**
```
NoC Flit (response) → NSU → AXI Slave
```
- Response flit 的 qos **繼承自對應 request**（與 02_flit.md 一致）
- NI 不修改 response qos

---

## 2. QoS Generator

位於 Initiator NI (NMU) 中，負責為每個 packet 產生 urgency level，寫入 flit header 的 `qos` 欄位。支援 3 種 generator 模式（加上 Bypass 直通）。

### 2.1 Operation Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Bypass** | 直接使用 AXI awqos/arqos，不做任何調整 | 預設行為，AXI Master 自行管理 QoS |
| **Fixed** | 軟體可程式化，為每個 packet 指定固定的 urgency level | 簡單系統，靜態優先級分配 |
| **Limiter** | 頻寬超過限制時，**阻塞 request**（暫停發送） | 硬性頻寬上限，防止單一 master 獨佔 |
| **Regulator** | 頻寬超過限制時，**降低 urgency 與 hurry 等級** | 軟性頻寬控制，動態優先級調節 |

### 2.2 Fixed Mode

最簡單的模式，為每個 packet 輸出軟體配置的固定 urgency level。

```
flit.hdr.qos = CSR.QOS_FIXED_VALUE;   // 所有 packet 使用相同 urgency
```

**CSR：**

| Register | Width | Reset | Description |
|----------|-------|-------|-------------|
| `QOS_MODE` | 2 | 0x0 | 0=Bypass, 1=Fixed, 2=Limiter, 3=Regulator |
| `QOS_FIXED_VALUE` | 4 | 0x0 | Fixed mode 輸出的 urgency level (0-15) |

### 2.3 Limiter Mode

**目的**：限制 Initiator 的頻寬上限。當實際頻寬超過配置的限制時，**阻塞後續 request**（暫停發送），直到頻寬回到限制以下。

> 這是**硬性頻寬限制** — request 被完全暫停，不只是降低優先級。

#### 2.3.1 Token Bucket 模型

```
                    ┌─────────────┐
   消耗 tokens ────▶│             │
   (發送 request)   │  Token      │◀──── 補充 tokens
                    │  Bucket     │      (每 cycle += BANDWIDTH_LIMIT)
                    │             │
                    └─────────────┘
                          │
                    tokens <= 0 ?
                     ╱         ╲
                   Yes          No
                    │            │
              ┌─────▼─────┐  ┌──▼──────────┐
              │  STALL    │  │ 正常發送     │
              │  Request  │  │ request     │
              └───────────┘  └─────────────┘
```

#### 2.3.2 Internal State

| State | Width | Type | Description |
|-------|-------|------|-------------|
| `token_counter` | BW_COUNTER_WIDTH (default 24) | Signed | Token 計數器，正值=有餘額，負值=透支 |

#### 2.3.3 運作流程

```
每個 Cycle：
  token_counter += BANDWIDTH_LIMIT;          // 補充 tokens（每 cycle 固定速率）
  token_counter = min(token_counter, BUCKET_DEPTH);  // 上限 clamp

當發送 Request 時：
  if (token_counter >= request_bytes)
      token_counter -= request_bytes;        // 消耗 tokens
      → 允許發送，flit.hdr.qos = QOS_FIXED_VALUE
  else
      → STALL request（暫停發送，直到 tokens 夠用）
```

> **Implementation Note：** Stall 實作為 NMU 的 injection buffer 停止向 NoC 注入新 flit。AXI Master 端會觀察到 awready/arready 被拉低。

#### 2.3.4 數值範例

**配置**：`BANDWIDTH_LIMIT=64`（每 cycle 補充 64 tokens），`BUCKET_DEPTH=512`

| Cycle | 事件 | tokens 變化 | tokens | 狀態 |
|-------|------|-------------|--------|------|
| 0 | 初始 | — | 512 | OK |
| 1 | 發送 256B request | 512 + 64 - 256 | 320 | OK |
| 2 | 發送 256B request | 320 + 64 - 256 | 128 | OK |
| 3 | 發送 256B request | 128 + 64 - 256 | -64 | **STALL** |
| 4 | 無（stall 中） | -64 + 64 | 0 | **STALL** |
| 5 | 無（stall 中） | 0 + 64 | 64 | **STALL** |
| 6 | 無（stall 中） | 64 + 64 | 128 | **STALL** |
| 7 | 無（stall 中） | 128 + 64 | 192 | **STALL** |
| 8 | 無（stall 中） | 192 + 64 | 256 | OK，可發送 |

> **解讀**：連續 3 筆 256B request 消耗完 tokens，Cycle 3-7 被 stall，Cycle 8 累積足夠 tokens 後恢復。

#### 2.3.5 CSR

| Register | Width | Reset | Description |
|----------|-------|-------|-------------|
| `BANDWIDTH_LIMIT` | 16 | 0x0000 | 每 cycle 補充的 token 數量（bytes/cycle） |
| `BUCKET_DEPTH` | 16 | 0x0200 | Token bucket 最大容量（bytes） |
| `QOS_FIXED_VALUE` | 4 | 0x0 | Limiter 允許發送時使用的 urgency level |

### 2.4 Regulator Mode

**目的**：軟性頻寬控制。當 Initiator 的實際頻寬超過配置的限制時，**降低 urgency 與 hurry 等級**，使該 Initiator 在 Router arbitration 中競爭力下降，從而間接減少其頻寬佔用。

> 這是**軟性頻寬限制** — request 仍然發送，但優先級降低。與 Limiter（硬性阻塞）形成互補。

#### 2.4.1 核心概念：Leaky Bucket

```
                    ┌─────────────┐
   實際發送量 ─────▶│             │──────▶ 水位 = bandwidth_counter
   (request bytes)  │  Leaky      │
                    │  Bucket     │
   配額消耗 ◀──────│ ─ ─ ─ ─ ─ ─│◀────── 漏出速率 = BANDWIDTH_BUDGET
   (每 cycle 漏出)  └─────────────┘
```

| 水位狀態 | 意義 | 系統反應 |
|----------|------|----------|
| **水位 > 0** (正值) | 發送 > 配額 → 頻寬**超額** | 降低 urgency → 降低 QoS |
| **水位 < 0** (負值) | 發送 < 配額 → 頻寬**未滿** | 提升 urgency → 提升 QoS |
| **水位 ≈ 0** | 發送 ≈ 配額 → 達到目標 | 維持當前 urgency |

> **鋸齒行為**：由於 AXI transaction 的突發特性（burst），bandwidth_counter 會呈現鋸齒狀波動：發送 burst 時快速上升，等待期間穩定下降。urgency 隨之震盪，這是正常的閉環控制行為。

#### 2.4.2 Internal State

| State | Width | Type | Description |
|-------|-------|------|-------------|
| `bandwidth_counter` | BW_COUNTER_WIDTH (default 24) | Signed | 頻寬差額累計器（實際發送 - 配額），saturating |
| `urgency_level` | URGENCY_WIDTH (default 3) | Unsigned | 當前 urgency (0 ~ 2^W-1) |
| `hurry_level` | 1 | Unsigned | 快速回應旗標，頻寬嚴重超額時清除 |

> 三者皆為硬體內部狀態，由硬體自動維護，軟體不可直接存取。

#### 2.4.3 運作流程

```
┌───────────────────────────────────────────────────────────────────────┐
│                         每個 Cycle 執行                                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Step 1: 更新 bandwidth_counter                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  if (request_sent_this_cycle)                                   │   │
│  │      counter += request_bytes;      // 實際發送量               │   │
│  │  counter -= BANDWIDTH_BUDGET;       // 扣除配額 (每 cycle)      │   │
│  │  counter = clamp(counter, MIN_SAT, MAX_SAT);  // saturate      │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  Step 2: 調整 urgency_level 與 hurry_level                            │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  if (counter > URGENCY_RISE_THRESHOLD)     // 頻寬超額          │   │
│  │      urgency = max(urgency - URGENCY_STEP, 0);   // ↓ 降低     │   │
│  │  else if (counter < URGENCY_DROP_THRESHOLD) // 頻寬未滿         │   │
│  │      urgency = min(urgency + URGENCY_STEP, MAX_URGENCY);  // ↑ │   │
│  │                                                                 │   │
│  │  hurry_level = (counter > HURRY_THRESHOLD) ? 0 : 1;            │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  Step 3: 計算最終 QoS (僅在發送 Request 時)                           │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  raw_qos = BASE_QOS + urgency_level;                            │   │
│  │  clamped_qos = min(raw_qos, 15);      // 限制在 4-bit 範圍      │   │
│  │  final_qos = max(clamped_qos, SOCKET_QOS);  // 下限保護         │   │
│  │  flit.hdr.qos = final_qos;                                     │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

#### 2.4.4 數值範例

**配置**：`BANDWIDTH_BUDGET=100`, `BASE_QOS=8`, `URGENCY_STEP=1`, `URGENCY_WIDTH=3` (max=7), `URGENCY_RISE_THRESHOLD=0`, `URGENCY_DROP_THRESHOLD=0`

| Cycle | 事件 | counter 變化 | counter | urgency | QoS |
|-------|------|-------------|---------|---------|-----|
| 0 | 初始 | — | 0 | 4 | 12 |
| 1 | 發送 512B | 0 + 512 - 100 | +412 | 4 - 1 = 3 | 11 |
| 2 | 發送 256B | +412 + 256 - 100 | +568 | 3 - 1 = 2 | 10 |
| 3 | 無 | +568 - 100 | +468 | 2 - 1 = 1 | 9 |
| 4 | 無 | +468 - 100 | +368 | 1 - 1 = 0 | 8 |
| 5 | 無 | +368 - 100 | +268 | 0 (floor) | 8 |
| 6 | 無 | +268 - 100 | +168 | 0 | 8 |
| 7 | 無 | +168 - 100 | +68 | 0 | 8 |
| 8 | 無 | +68 - 100 | -32 | 0 + 1 = 1 | 9 |
| 9 | 發送 512B | -32 + 512 - 100 | +380 | 1 - 1 = 0 | 8 |

> **解讀**：Cycle 1-2 大量發送導致 counter 上升，urgency 逐步降低（QoS 從 12 降至 10）。Cycle 3-8 無發送，counter 慢慢漏出至負值，urgency 開始回升。Cycle 9 再次大量發送，counter 跳升，urgency 又降回。這就是典型的鋸齒行為。

#### 2.4.5 QoS Saturation

當 `BASE_QOS + urgency_level > 15` 時：

```
範例：BASE_QOS = 12, urgency_level = 7

raw_qos     = 12 + 7 = 19     // 超過 4-bit 最大值
clamped_qos = min(19, 15) = 15
final_qos   = max(15, SOCKET_QOS)
```

#### 2.4.6 CSR

| Register | Width | Reset | Description |
|----------|-------|-------|-------------|
| `BANDWIDTH_BUDGET` | 16 | 0x0000 | 頻寬配額（bytes/cycle），每 cycle 從 counter 扣除 |
| `BASE_QOS` | 4 | 0x0 | 基礎 urgency（urgency=0 時的 QoS 值） |
| `URGENCY_STEP` | 2 | 0x1 | Urgency 每次調整的步進值 (1~3) |
| `URGENCY_RISE_THRESHOLD` | 16 | 0x0000 | counter 超過此值時降低 urgency |
| `URGENCY_DROP_THRESHOLD` | 16 | 0x0000 | counter 低於此值時提升 urgency |
| `HURRY_THRESHOLD` | 16 | 0x0000 | counter 超過此值時清除 hurry flag |
| `SOCKET_QOS_EN` | 1 | 0x0 | 啟用 Socket QoS 下限保護 |
| `SOCKET_QOS` | 4 | 0x0 | Socket QoS 下限值 |

> **SOCKET_QOS 用途**：確保 QoS 不會低於某個下限，即使 urgency=0 也能維持最低優先級。

> **Implementation Note：** `bandwidth_counter` 使用 saturating signed arithmetic。C++ 實作時需自行實作 `sat_add` / `sat_sub`，避免 undefined behavior。

### 2.5 Limiter vs Regulator 比較

| 特性 | Limiter | Regulator |
|------|---------|-----------|
| **控制方式** | 硬性：**阻塞 request** | 軟性：**降低 urgency** |
| **頻寬超額時** | 暫停發送，等 token 回復 | 仍然發送，但優先級降低 |
| **對 AXI Master 的影響** | awready/arready 拉低 | 無影響，Master 無感知 |
| **延遲影響** | 可能增加延遲（stall） | 不直接增加延遲 |
| **典型用途** | 嚴格頻寬上限 | 軟性頻寬控制，避免獨佔 |
| **組合使用** | 可同時啟用（先 Regulator 降級，再 Limiter 阻塞） | — |

### 2.6 QoS 決策流程

```
                    ┌─────────────┐
                    │ AXI Request │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  QOS_MODE?  │
                    └──────┬──────┘
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────▼─────┐     ┌─────▼─────┐     ┌──────▼──────┐
  │   Fixed   │     │  Limiter  │     │  Regulator  │
  │ urgency   │     │  (token   │     │  (urgency   │
  │ = fixed   │     │   bucket) │     │   adjust)   │
  └─────┬─────┘     └─────┬─────┘     └──────┬──────┘
        │                 │                   │
        │           ┌─────▼─────┐     ┌───────▼───────┐
        │           │ tokens    │     │ Calculate     │
        │           │ enough?   │     │ urgency/hurry │
        │           └──┬────┬──┘     └───────┬───────┘
        │           No │  Yes│               │
        │         STALL│    │               │
        │              │    │               │
        ▼              ▼    ▼               ▼
  ┌─────────────────────────────────────────────┐
  │          Final Urgency → qos[3:0]           │
  └──────────────────┬──────────────────────────┘
                     │
              ┌──────▼──────┐
              │ Flit Header │
              │   qos[3:0]  │
              └─────────────┘
```

---

## 3. Router Arbitration Schemes

Router 的 Switch Allocator 支援 4 種 arbitration scheme，由 CSR 配置選擇。Arbitration 決定當多個 input port 競爭同一 output port 時，哪個 flit 優先通過。

### 3.1 Scheme Overview

| Scheme | ID | Description | 特性 |
|--------|----|-------------|------|
| **FIFO** | 0 | 先到先服務，依 request 到達順序 | 預設，公平但不考慮優先級 |
| **Fixed Priority** | 1 | 靜態優先級，依 port 編號（可配置） | 簡單但可能 starvation |
| **Round Robin** | 2 | 輪流服務各 requesting port | 公平，無 starvation |
| **Rotate** | 3 | 旋轉優先級，每次 grant 後最高優先級輪轉 | 結合 Fixed 與 RR 優點 |

### 3.2 FIFO (Default)

依 flit 到達 input buffer 的先後順序服務。

```
Arrival order: Port2(t=5) → Port0(t=7) → Port1(t=9)
Grant order:   Port2 → Port0 → Port1
```

**特性：**
- 不考慮 qos 值，純粹依時間順序
- 天然公平，無 starvation 風險
- 適用於所有 traffic 同等重要的場景

### 3.3 Fixed Priority

每個 input port 有靜態優先級，高優先級 port 永遠優先。

```
Priority: Port0 > Port1 > Port2 > Port3 > Local
（可由 CSR 配置優先級順序）
```

**特性：**
- 最低延遲給高優先級 port
- 低優先級 port 可能 starvation
- 適用於有明確優先級需求的場景

### 3.4 Round Robin

在所有 requesting port 之間輪流服務。

```
Cycle 1: Grant Port0 → pointer moves to Port1
Cycle 2: Grant Port1 → pointer moves to Port2
Cycle 3: Port2 no request, skip → Grant Port3
```

**特性：**
- 完全公平，保證每個 port 最終都會被服務
- 無 starvation
- 不考慮 qos 值

### 3.5 Rotate (Rotating Priority)

結合 Fixed Priority 與 Round Robin：有優先級順序，但每次 grant 後最高優先級旋轉到下一個 port。

```
初始:   Port0 > Port1 > Port2 > Port3
Grant Port0 後: Port1 > Port2 > Port3 > Port0
Grant Port1 後: Port2 > Port3 > Port0 > Port1
```

**特性：**
- 短期有優先級差異（有利於低延遲 traffic）
- 長期趨於公平（旋轉保證所有 port 輪流成為最高優先級）
- 可視為「帶優先級的 Round Robin」

### 3.6 Wormhole 交互

無論使用哪種 arbitration scheme，wormhole switching 的 path lock 規則不變：

- Arbitration **僅在 HEAD flit** 發生
- Body/Tail flit 跟隨 HEAD 已鎖定的 path，**不參與重新 arbitration**
- 高優先級 flit 無法 preempt 已鎖定的低優先級 wormhole path（Priority Inversion）
- 緩解方式：限制 AXI burst length（縮短 blocking 時間）

### 3.7 QoS 欄位與 Arbitration 的關係

`qos` 欄位是否被 arbitration 使用，取決於 scheme：

| Scheme | 是否使用 qos 欄位 | 說明 |
|--------|-------------------|------|
| FIFO | 否 | 純粹依時間順序 |
| Fixed Priority | 否 | 依 port 靜態優先級 |
| Round Robin | 否 | 依輪轉順序 |
| Rotate | 否 | 依旋轉優先級 |

> **注意**：4 種 scheme 均不直接使用 flit header 的 `qos` 欄位。`qos` 欄位的作用是讓 NI 的 QoS Generator（Limiter/Regulator）進行頻寬控制，並提供 Probe 統計依據。未來可擴展 QoS-Aware scheme（結合 qos 值與 RR 的混合策略）。

---

## 4. Performance Probes

NI 中的 Probe Unit 負責收集效能與錯誤資訊。支援 3 種 probe 類型。

### 4.1 Probe Types

| Probe | 測量內容 | 用途 |
|-------|---------|------|
| **Error Probe** | 協議錯誤、ECC 錯誤 | 錯誤偵測與記錄 |
| **Packet Probe** | 頻寬 (bytes/time) | 頻寬監控、瓶頸分析 |
| **Transaction Probe** | 延遲 (cycles) | 延遲分析、SLA 驗證 |

### 4.2 Error Probe

偵測並記錄傳輸錯誤。

**偵測的錯誤類型：**

| Error Type | 偵測方式 | 嚴重程度 |
|------------|---------|----------|
| ECC Uncorrectable | SECDED 解碼失敗 | FATAL |
| Timeout | Response 超時未回 | ERROR |
| Protocol Violation | 非預期的 flit 格式 | ERROR |

**Error Probe 行為：**
- 偵測到錯誤時，設置 `ERR_STATUS` 對應 bit
- `ERR_COUNT` 自動 +1（saturating，不 wrap-around）
- `LAST_ERR_INFO` 記錄最近一次錯誤的相關資訊（AXI ID, src_id, dst_id）
- 錯誤旗標由軟體寫 1 清除（Write-1-to-Clear）

### 4.3 Packet Probe (Bandwidth)

統計一段時間內的傳輸量。

**功能：**
- 可選擇統計 Read、Write 或 Combined
- 可配置統計視窗大小
- 支援 snapshot 和 continuous 模式

**CSR：**

| Register | Width | Reset | Access | Description |
|----------|-------|-------|--------|-------------|
| `PKT_PROBE_EN` | 1 | 0x0 | RW | 啟用 Packet Probe |
| `PKT_PROBE_MODE` | 2 | 0x0 | RW | 0=Combined, 1=Read, 2=Write |
| `PKT_WINDOW_SIZE` | 16 | 0x0000 | RW | 統計視窗 (cycles) |
| `PKT_BYTE_COUNT` | 32 | 0x0 | RO | 傳輸 bytes 計數 |
| `PKT_BANDWIDTH` | 32 | 0x0 | RO | 計算的頻寬 |

### 4.4 Transaction Probe (Latency)

統計 transaction 延遲分布。

**功能：**
- 測量 request 發出到 response 回來的 cycles
- 將延遲分成 N+1 個區間，統計落入各區間的 transaction 數量
- 可配置區間邊界

**延遲區間：**
```
Bin 0: latency < THRESHOLD_0
Bin 1: THRESHOLD_0 <= latency < THRESHOLD_1
Bin 2: THRESHOLD_1 <= latency < THRESHOLD_2
...
Bin N: latency >= THRESHOLD_{N-1}
```

**CSR：**

| Register | Width | Reset | Access | Description |
|----------|-------|-------|--------|-------------|
| `TXN_PROBE_EN` | 1 | 0x0 | RW | 啟用 Transaction Probe |
| `TXN_THRESHOLD_0` | 16 | 0x0000 | RW | 區間邊界 0 (cycles) |
| `TXN_THRESHOLD_1` | 16 | 0x0000 | RW | 區間邊界 1 (cycles) |
| `TXN_THRESHOLD_2` | 16 | 0x0000 | RW | 區間邊界 2 (cycles) |
| `TXN_THRESHOLD_3` | 16 | 0x0000 | RW | 區間邊界 3 (cycles) |
| `TXN_BIN_0_COUNT` | 32 | 0x0 | RO | Bin 0 計數 |
| `TXN_BIN_1_COUNT` | 32 | 0x0 | RO | Bin 1 計數 |
| `TXN_BIN_2_COUNT` | 32 | 0x0 | RO | Bin 2 計數 |
| `TXN_BIN_3_COUNT` | 32 | 0x0 | RO | Bin 3 計數 |
| `TXN_BIN_4_COUNT` | 32 | 0x0 | RO | Bin 4 計數 |
| `TXN_MIN_LATENCY` | 16 | 0xFFFF | RO | 最小延遲 |
| `TXN_MAX_LATENCY` | 16 | 0x0000 | RO | 最大延遲 |
| `TXN_TOTAL_COUNT` | 32 | 0x0 | RO | 總 transaction 數 |

---

## 5. NI CSR Memory Map

### 5.1 Register Summary

| Offset | Register | Access | Reset | Description |
|--------|----------|--------|-------|-------------|
| **QoS Generator** |||||
| 0x000 | `QOS_MODE` | RW | 0x0 | QoS 模式選擇 (2-bit) |
| 0x004 | `QOS_FIXED_VALUE` | RW | 0x0 | Fixed mode urgency level (4-bit) |
| 0x008 | `BANDWIDTH_LIMIT` | RW | 0x0000 | Limiter: token 補充速率 (16-bit) |
| 0x00C | `BUCKET_DEPTH` | RW | 0x0200 | Limiter: token bucket 最大容量 (16-bit) |
| 0x010 | `BANDWIDTH_BUDGET` | RW | 0x0000 | Regulator: 頻寬配額 (16-bit) |
| 0x014 | `BASE_QOS` | RW | 0x0 | Regulator: 基礎 urgency (4-bit) |
| 0x018 | `URGENCY_STEP` | RW | 0x1 | Regulator: urgency 步進值 (2-bit) |
| 0x01C | `URGENCY_RISE_THRESHOLD` | RW | 0x0000 | Regulator: urgency 下降門檻 (16-bit) |
| 0x020 | `URGENCY_DROP_THRESHOLD` | RW | 0x0000 | Regulator: urgency 上升門檻 (16-bit) |
| 0x024 | `HURRY_THRESHOLD` | RW | 0x0000 | Regulator: hurry 門檻 (16-bit) |
| 0x028 | `SOCKET_QOS_EN` | RW | 0x0 | Socket QoS 啟用 (1-bit) |
| 0x02C | `SOCKET_QOS` | RW | 0x0 | Socket QoS 下限值 (4-bit) |
| **Arbitration** |||||
| 0x030 | `ARB_SCHEME` | RW | 0x0 | Arbitration scheme (2-bit, per-router) |
| 0x034 | `ARB_FIXED_PRIORITY` | RW | 0x0 | Fixed priority 順序 (port map) |

> **Note：** `ARB_SCHEME` 與 `ARB_FIXED_PRIORITY` 邏輯上屬於 Router 配置，但放置於 NI CSR address space 以實現統一的軟體管理介面。每個 NI 的 ARB registers 對應其連接的 Router。

| **Packet Probe** |||||
| 0x040 | `PKT_PROBE_EN` | RW | 0x0 | 啟用 Packet Probe |
| 0x044 | `PKT_PROBE_MODE` | RW | 0x0 | 統計模式 |
| 0x048 | `PKT_WINDOW_SIZE` | RW | 0x0000 | 統計視窗大小 |
| 0x04C | `PKT_BYTE_COUNT` | RO | 0x0 | Byte 計數 |
| 0x050 | `PKT_BANDWIDTH` | RO | 0x0 | 頻寬計算結果 |
| **Transaction Probe** |||||
| 0x060 | `TXN_PROBE_EN` | RW | 0x0 | 啟用 Transaction Probe |
| 0x064 | `TXN_THRESHOLD_0` | RW | 0x0000 | 延遲閾值 0 |
| 0x068 | `TXN_THRESHOLD_1` | RW | 0x0000 | 延遲閾值 1 |
| 0x06C | `TXN_THRESHOLD_2` | RW | 0x0000 | 延遲閾值 2 |
| 0x070 | `TXN_THRESHOLD_3` | RW | 0x0000 | 延遲閾值 3 |
| 0x080 | `TXN_BIN_0_COUNT` | RO | 0x0 | Bin 0 計數 |
| 0x084 | `TXN_BIN_1_COUNT` | RO | 0x0 | Bin 1 計數 |
| 0x088 | `TXN_BIN_2_COUNT` | RO | 0x0 | Bin 2 計數 |
| 0x08C | `TXN_BIN_3_COUNT` | RO | 0x0 | Bin 3 計數 |
| 0x090 | `TXN_BIN_4_COUNT` | RO | 0x0 | Bin 4 計數 |
| 0x094 | `TXN_MIN_LATENCY` | RO | 0xFFFF | 最小延遲 |
| 0x098 | `TXN_MAX_LATENCY` | RO | 0x0000 | 最大延遲 |
| 0x09C | `TXN_TOTAL_COUNT` | RO | 0x0 | 總 transaction 數 |
| **Error Probe** |||||
| 0x100 | `ERR_STATUS` | W1C | 0x0 | 錯誤狀態 (Write-1-to-Clear) |
| 0x104 | `ERR_COUNT` | RO | 0x0 | 錯誤計數 (saturating) |
| 0x108 | `ECC_UNCORR_ERR_CNT` | RO | 0x0 | ECC Uncorrectable 錯誤計數 (saturating) |
| 0x10C | `LAST_ERR_INFO` | RO | 0x0 | 最近錯誤資訊 |

### 5.2 QOS_MODE Register (0x000)

| Field | Bit | Reset | Description |
|-------|-----|-------|-------------|
| `mode` | [1:0] | 0x0 | 0=Bypass, 1=Fixed, 2=Limiter, 3=Regulator |
| Reserved | [31:2] | — | — |

### 5.3 ARB_SCHEME Register (0x030)

| Field | Bit | Reset | Description |
|-------|-----|-------|-------------|
| `scheme` | [1:0] | 0x0 | 0=FIFO, 1=Fixed, 2=RoundRobin, 3=Rotate |
| Reserved | [31:2] | — | — |

### 5.4 ERR_STATUS Register (0x100)

| Field | Bit | Access | Description |
|-------|-----|--------|-------------|
| `ecc_uncorr_err` | [0] | W1C | ECC Uncorrectable 錯誤發生 |
| `timeout_err` | [1] | W1C | Timeout 錯誤發生 |
| `protocol_err` | [2] | W1C | Protocol violation 錯誤發生 |
| Reserved | [31:3] | — | — |

### 5.5 LAST_ERR_INFO Register (0x10C)

| Field | Bit | Width | Description |
|-------|-----|-------|-------------|
| `err_axi_id` | [7:0] | 8 | 錯誤 transaction 的 AXI ID |
| `err_src_id` | [15:8] | 8 | 錯誤來源 node ID |
| `err_dst_id` | [23:16] | 8 | 錯誤目標 node ID |
| Reserved | [31:24] | 8 | — |

### 5.6 Error Counter Behavior

所有 error counter 使用 **saturating arithmetic**：

| Counter | Width | Saturation | Clear |
|---------|-------|------------|-------|
| `ERR_COUNT` | ERR_COUNTER_WIDTH (default 16) | 2^W - 1 | Write 1 to ERR_STATUS |
| `ECC_UNCORR_ERR_CNT` | ERR_COUNTER_WIDTH (default 16) | 2^W - 1 | Write 1 to ERR_STATUS[0] |

```
if (counter < MAX_VALUE)
    counter = counter + 1;
// else: counter stays at MAX_VALUE (no wrap-around)
```

---

## 6. Design Considerations

### 6.1 QoS vs Starvation

高優先級 traffic 不應造成低優先級 traffic 永久 starvation：

| 機制 | 說明 |
|------|------|
| Round Robin / Rotate scheme | Arbitration 本身保證公平性 |
| Regulator mode | 高頻寬 initiator 自動降低 urgency |
| Separate VC per QoS class | 不同 QoS 使用不同 Virtual Channel（未來擴展） |

### 6.2 Error Response QoS

Response flit 的 qos **繼承自對應 request**，不因錯誤而修改：

```
response.qos = original_request.qos;  // 無論 bresp 為何
```

### 6.3 QoS 與 Flow Control

QoS 機制（Generator + Arbitration）與 Flow Control（credit/ready）獨立運作：

- QoS 影響 **arbitration 決策**（誰優先通過 crossbar）
- Flow Control 影響 **是否可以發送**（下游是否有空間）
- 即使 qos=15，若下游 credit=0 或 ready=0，仍無法發送
- Flow Control 正確性優先於 QoS 優先級

---

## 7. Integration Notes

### 7.1 QoS 計算時機

| 問題 | 規則 |
|------|------|
| 計算位置 | NMU 內的 QoS Generator |
| 計算時機 | 產生 AW/AR flit 時 |
| W flit QoS | 繼承對應 AW flit 的 qos |
| Response QoS | NSU 從 Request Info Store 讀取原始 request qos，直接繼承 |

### 7.2 Limiter Stall 機制

Limiter stall 的實作方式：

```
NMU Injection Buffer:
  if (QOS_MODE == Limiter && token_counter < next_request_bytes)
      injection_valid = 0;     // 停止注入 flit
      axi_awready = 0;         // 停止接受 AXI request
      axi_arready = 0;
```

> stall 期間 token_counter 持續補充，直到累積足夠 tokens 後恢復。

### 7.3 Regulator 的 Bandwidth Counter 來源

Regulator 追蹤的是**發送的 request bytes**（ReqPath），而非 response bytes：

```
每發送一筆 AW/AR request：
    counter += (awlen + 1) * (DATA_WIDTH / 8);   // AW: burst 的總 bytes
    counter += (DATA_WIDTH / 8);                   // AR: 單次 read bytes
```

### 7.4 Arbitration Scheme 配置範圍

`ARB_SCHEME` 為 **per-router** 配置，同一 router 所有 output port 使用相同 scheme。不支援 per-output 不同 scheme。

---

## Related Documents

- [Flit Format](02_flit.md) — Header qos 欄位定義、NODE_ID_WIDTH、QOS_WIDTH
- [Router](03_router.md) — Router Switch Allocator、arbitration 框架
- [Network Interface](04_network_interface.md) — NI 架構、NMU/NSU 功能區塊

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
| 2.0 | 2026-03-15 | Limiter 改為阻塞式 (token bucket)；Regulator 改為降低 urgency；新增 4 種 arbitration scheme；新增 Error Probe；完整 register table 含 reset value |
