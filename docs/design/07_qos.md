# QoS and Performance Monitoring

NoC 的 Quality of Service (QoS) 機制與效能監控設計。

---

## 1. Overview

QoS 機制確保不同 traffic 類型獲得適當的頻寬和延遲保證。

QoS 機制整合 QoS Generator、Probe 架構與 flit header qos 欄位等設計，形成自有的混合設計。QoS 相關欄位寬度與 [02_flit.md](02_flit.md) 保持一致。

### 1.0 QoS 相關參數

#### 來自 Flit Format 的參數

| Parameter | Range | Source |
|-----------|-------|--------|
| `NODE_ID_WIDTH` | 8 (Fixed) | 02_flit.md |
| `QOS_WIDTH` | 4 (Fixed) | 02_flit.md |

#### QoS 模組專用參數

| Parameter | Range | Description |
|-----------|-------|-------------|
| `BW_COUNTER_WIDTH` | 16 ~ 32 | Bandwidth counter 寬度 |
| `URGENCY_WIDTH` | 2 ~ 4 | Urgency level 寬度 |
| `ERR_COUNTER_WIDTH` | 8 ~ 32 | Error counter 寬度 |

### 1.1 QoS 層級

| Layer | Component | 功能 |
|-------|-----------|------|
| Flit | Header `qos` (4 bits) | 攜帶優先級資訊，Router arbitration 使用 |
| NI | QoS Generator | 產生/調整 qos 值 |
| NI | QoS Probe | 統計頻寬、延遲 |
| System | CSR Interface | 軟體配置與監控 |

### 1.2 與 Flit Format 關聯

Header 中的 `qos` 欄位（4 bits, 16 levels）處理方式：

**Request Path (NMU)：**
```
AXI Master (awqos/arqos) → NMU (QoS Generator) → NoC Flit (qos in header)
```
- `QOS_MODE = Bypass`：直接使用 AXI awqos/arqos（預設行為，與 02_flit.md 描述一致）
- `QOS_MODE = Fixed/Limiter/Regulator`：由 QoS Generator 產生或調整

**Response Path (NSU)：**
```
NoC Flit (response) → NSU → AXI Slave
```
- Response flit 的 qos **繼承自對應 request**（與 02_flit.md 一致）
- NI 不修改 response qos

---

## 2. QoS Generator

NI 中的 QoS Generator 負責產生或調整 flit header 的 `qos` 值。

### 2.1 Operation Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Fixed** | 固定輸出指定的 qos 值 | 簡單系統，靜態優先級 |
| **Limiter** | 限制頻寬上限，超過時降低 qos | 防止單一 master 獨佔頻寬 |
| **Regulator** | 動態調節，保證最低頻寬 | 需要頻寬保證的 real-time traffic |

### 2.2 Fixed Mode

最簡單的模式，直接輸出配置的 qos 值。

```
if (FIXED_MODE)
    flit.hdr.qos = CSR.QOS_FIXED_VALUE;
```

**CSR:**

| Register | Width | Description |
|----------|-------|-------------|
| `QOS_MODE` | 2 | 0=Bypass, 1=Fixed, 2=Limiter, 3=Regulator |
| `QOS_FIXED_VALUE` | 4 | Fixed mode 輸出的 qos 值 (0-15) |

### 2.3 Limiter Mode

限制頻寬上限，當使用量超過閾值時降低 qos。

**Internal State:**

| State | Width | Description |
|-------|-------|-------------|
| `bandwidth_counter` | BW_COUNTER_WIDTH (16~32, default 24) | 累計頻寬計數器 (signed) |

**原理：**
```
bandwidth_counter [BW_COUNTER_WIDTH-1:0]:  // signed, saturating arithmetic
  - 每發送一筆 request，counter = sat_add(counter, request_bytes)
  - 每 cycle，counter = sat_sub(counter, BANDWIDTH_LIMIT)
  - Saturation: clamp to [-(2^(W-1)), 2^(W-1)-1]

if (counter > SATURATION_THRESHOLD)
    flit.hdr.qos = LOW_PRIORITY;
else
    flit.hdr.qos = CSR.QOS_FIXED_VALUE;
```

**CSR:**

| Register | Width | Description |
|----------|-------|-------------|
| `BANDWIDTH_LIMIT` | 16 | 頻寬限制 (1/256 bytes/cycle) |
| `SATURATION_THRESHOLD` | 16 | 飽和閾值 (bytes) |
| `LOW_PRIORITY` | 4 | 超過閾值時使用的低優先級 |

### 2.4 Regulator Mode

**目的**：保證 Master 獲得最低頻寬。當實際頻寬低於目標時，自動提升 QoS 以爭取更多網路資源。

#### 2.4.1 核心概念：水桶模型

可以把 Regulator 想像成一個「漏水的水桶」：

```
                    ┌─────────────┐
   實際頻寬 ──────▶ │             │ ──────▶ 水位 = bandwidth_counter
   (Response)       │   水 桶     │
                    │             │
   目標頻寬 ◀────── │ ─ ─ ─ ─ ─ ─│ ◀────── 漏水速率 = BANDWIDTH_BUDGET
   (每 cycle 漏出)  └─────────────┘
```

| 水位狀態 | 意義 | 系統反應 |
|----------|------|----------|
| **水位 < 0** (負值) | 漏出 > 進入 → 實際頻寬**不足** | 提高 urgency → 提高 QoS |
| **水位 > 0** (正值) | 進入 > 漏出 → 實際頻寬**充足** | 降低 urgency → 降低 QoS |
| **水位 ≈ 0** | 進出平衡 → 達到目標頻寬 | 維持當前 urgency |

#### 2.4.2 內部狀態

| State | Width | Type | Description |
|-------|-------|------|-------------|
| `bandwidth_counter` | BW_COUNTER_WIDTH (default 24) | Signed | 頻寬差額累計器 (實際 - 目標) |
| `urgency_level` | URGENCY_WIDTH (default 3) | Unsigned | 緊急程度 (0 ~ 7) |

> **兩者皆為硬體內部狀態**，由硬體自動維護，軟體不可直接存取。

#### 2.4.3 運作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                        每個 Cycle 執行                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: 更新 bandwidth_counter                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  if (response_received)                                      │   │
│  │      counter += response_bytes;   // 實際獲得的頻寬          │   │
│  │  counter -= BANDWIDTH_BUDGET;     // 扣除目標頻寬 (每 cycle) │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Step 2: 根據 counter 調整 urgency_level                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  if (counter < 0)         // 頻寬不足                        │   │
│  │      urgency = min(urgency + URGENCY_STEP, MAX_URGENCY);     │   │
│  │  else if (counter > 0)    // 頻寬充足                        │   │
│  │      urgency = max(urgency - URGENCY_STEP, 0);               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  Step 3: 計算最終 QoS (僅在發送 Request 時)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  raw_qos = BASE_QOS + urgency_level;                         │   │
│  │  clamped_qos = min(raw_qos, 15);    // 限制在 4-bit 範圍     │   │
│  │  final_qos = max(clamped_qos, SOCKET_QOS);                   │   │
│  │  flit.hdr.qos = final_qos;                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2.4.4 數值範例

**配置**：`BANDWIDTH_BUDGET=100`, `BASE_QOS=4`, `URGENCY_STEP=1`, `URGENCY_WIDTH=3` (max=7)

| Cycle | Response | counter 變化 | counter | urgency 變化 | urgency | QoS |
|-------|----------|--------------|---------|--------------|---------|-----|
| 0 | - | 初始 | 0 | 初始 | 0 | 4 |
| 1 | 無 | 0 - 100 | -100 | 0 + 1 | 1 | 5 |
| 2 | 無 | -100 - 100 | -200 | 1 + 1 | 2 | 6 |
| 3 | 無 | -200 - 100 | -300 | 2 + 1 | 3 | 7 |
| 4 | 512B | -300 + 512 - 100 | +112 | 3 - 1 | 2 | 6 |
| 5 | 256B | +112 + 256 - 100 | +268 | 2 - 1 | 1 | 5 |
| 6 | 無 | +268 - 100 | +168 | 1 - 1 | 0 | 4 |

> **解讀**：Cycle 1-3 沒有收到 response，系統認為頻寬不足，逐步提高 QoS。Cycle 4-6 收到 response，頻寬恢復，QoS 逐步降回基準值。

#### 2.4.5 QoS Saturation

當 `BASE_QOS + urgency_level > 15` 時，需要 saturation：

```
範例：BASE_QOS = 12, urgency_level = 7

raw_qos     = 12 + 7 = 19    // 超過 4-bit 最大值
clamped_qos = min(19, 15) = 15
final_qos   = max(15, SOCKET_QOS)
```

#### 2.4.6 CSR 配置

| Register | Width | Description |
|----------|-------|-------------|
| `BANDWIDTH_BUDGET` | 16 | 目標頻寬，單位 1/256 bytes/cycle |
| `BASE_QOS` | 4 | 基礎 QoS 值 (urgency=0 時使用) |
| `URGENCY_STEP` | 2 | Urgency 每次調整的步進值 (1~3) |
| `SOCKET_QOS_EN` | 1 | 啟用 Socket QoS 下限保護 |
| `SOCKET_QOS` | 4 | Socket QoS 下限值 |

> **SOCKET_QOS 用途**：確保 QoS 不會低於某個下限，即使 urgency=0 也能維持最低優先級。

#### 2.4.7 與 Limiter Mode 的比較

| 特性 | Limiter Mode | Regulator Mode |
|------|--------------|----------------|
| **目的** | 限制頻寬上限 | 保證頻寬下限 |
| **觸發條件** | 使用量**超過**閾值 | 使用量**低於**預算 |
| **QoS 調整方向** | 超標時**降低** QoS | 不足時**提升** QoS |
| **典型用途** | 防止單一 Master 獨佔 | Real-time traffic 保證 |
| **urgency 機制** | 無 | 有（漸進式調整） |

### 2.5 QoS 決策流程

```
                    ┌─────────────┐
                    │ AXI Request │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  QOS_MODE?  │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
     │   Fixed   │   │  Limiter  │   │ Regulator │
     └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
           │               │               │
           │         ┌─────▼─────┐   ┌─────▼─────┐
           │         │ counter > │   │ Calculate │
           │         │ threshold?│   │  urgency  │
           │         └─────┬─────┘   └─────┬─────┘
           │               │               │
           ▼               ▼               ▼
     ┌─────────────────────────────────────────┐
     │            Final QoS Value              │
     └─────────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ Flit Header │
                    │   qos[3:0]  │
                    └─────────────┘
```

---

## 3. Performance Probe

NI 中的 Probe 負責統計效能指標。

### 3.1 Probe Types

| Probe | 測量內容 | 用途 |
|-------|---------|------|
| **Packet Probe** | 頻寬 (bytes/time) | 頻寬監控、瓶頸分析 |
| **Transaction Probe** | 延遲 (cycles) | 延遲分析、SLA 驗證 |

### 3.2 Packet Probe (Bandwidth)

統計一段時間內的傳輸量。

**功能：**
- 可選擇統計 Read、Write 或 Combined
- 可配置統計視窗大小
- 支援 snapshot 和 continuous 模式

**CSR:**

| Register | Width | Description |
|----------|-------|-------------|
| `PKT_PROBE_EN` | 1 | 啟用 Packet Probe |
| `PKT_PROBE_MODE` | 2 | 0=Combined, 1=Read, 2=Write |
| `PKT_WINDOW_SIZE` | 16 | 統計視窗 (cycles) |
| `PKT_BYTE_COUNT` | 32 | 傳輸 bytes 計數 (RO) |
| `PKT_BANDWIDTH` | 32 | 計算的頻寬 (RO) |

### 3.3 Transaction Probe (Latency)

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

**CSR:**

| Register | Width | Description |
|----------|-------|-------------|
| `TXN_PROBE_EN` | 1 | 啟用 Transaction Probe |
| `TXN_THRESHOLD_0` | 16 | 區間邊界 0 (cycles) |
| `TXN_THRESHOLD_1` | 16 | 區間邊界 1 (cycles) |
| `TXN_THRESHOLD_2` | 16 | 區間邊界 2 (cycles) |
| `TXN_THRESHOLD_3` | 16 | 區間邊界 3 (cycles) |
| `TXN_BIN_0_COUNT` | 32 | Bin 0 計數 (RO) |
| `TXN_BIN_1_COUNT` | 32 | Bin 1 計數 (RO) |
| `TXN_BIN_2_COUNT` | 32 | Bin 2 計數 (RO) |
| `TXN_BIN_3_COUNT` | 32 | Bin 3 計數 (RO) |
| `TXN_BIN_4_COUNT` | 32 | Bin 4 計數 (RO) |
| `TXN_MIN_LATENCY` | 16 | 最小延遲 (RO) |
| `TXN_MAX_LATENCY` | 16 | 最大延遲 (RO) |
| `TXN_TOTAL_COUNT` | 32 | 總 transaction 數 (RO) |

---

## 4. NI CSR Memory Map

### 4.1 Register Summary

| Offset | Register | Access | Description |
|--------|----------|--------|-------------|
| **QoS Generator** ||||
| 0x000 | `QOS_MODE` | RW | QoS 模式選擇 |
| 0x004 | `QOS_FIXED_VALUE` | RW | Fixed mode qos 值 |
| 0x008 | `BANDWIDTH_LIMIT` | RW | Limiter 頻寬限制 |
| 0x00C | `SATURATION_THRESHOLD` | RW | 飽和閾值 |
| 0x010 | `LOW_PRIORITY` | RW | 低優先級值 |
| 0x014 | `BANDWIDTH_BUDGET` | RW | Regulator 頻寬預算 |
| 0x018 | `BASE_QOS` | RW | 基礎 qos 值 |
| 0x01C | `SOCKET_QOS_EN` | RW | Socket QoS 啟用 |
| 0x020 | `SOCKET_QOS` | RW | Socket QoS 值 |
| **Packet Probe** ||||
| 0x040 | `PKT_PROBE_EN` | RW | 啟用 Packet Probe |
| 0x044 | `PKT_PROBE_MODE` | RW | 統計模式 |
| 0x048 | `PKT_WINDOW_SIZE` | RW | 統計視窗大小 |
| 0x04C | `PKT_BYTE_COUNT` | RO | Byte 計數 |
| 0x050 | `PKT_BANDWIDTH` | RO | 頻寬計算結果 |
| **Transaction Probe** ||||
| 0x060 | `TXN_PROBE_EN` | RW | 啟用 Transaction Probe |
| 0x064 | `TXN_THRESHOLD_0` | RW | 延遲閾值 0 |
| 0x068 | `TXN_THRESHOLD_1` | RW | 延遲閾值 1 |
| 0x06C | `TXN_THRESHOLD_2` | RW | 延遲閾值 2 |
| 0x070 | `TXN_THRESHOLD_3` | RW | 延遲閾值 3 |
| 0x080 | `TXN_BIN_0_COUNT` | RO | Bin 0 計數 |
| 0x084 | `TXN_BIN_1_COUNT` | RO | Bin 1 計數 |
| 0x088 | `TXN_BIN_2_COUNT` | RO | Bin 2 計數 |
| 0x08C | `TXN_BIN_3_COUNT` | RO | Bin 3 計數 |
| 0x090 | `TXN_BIN_4_COUNT` | RO | Bin 4 計數 |
| 0x094 | `TXN_MIN_LATENCY` | RO | 最小延遲 |
| 0x098 | `TXN_MAX_LATENCY` | RO | 最大延遲 |
| 0x09C | `TXN_TOTAL_COUNT` | RO | 總 transaction 數 |
| **Error Status** ||||
| 0x100 | `ERR_STATUS` | RO | 錯誤狀態 |
| 0x104 | `ERR_COUNT` | RO | 錯誤計數 (ERR_COUNTER_WIDTH, saturating) |
| 0x108 | `ECC_UNCORR_ERR_CNT` | RO | ECC Uncorrectable 錯誤計數 (ERR_COUNTER_WIDTH, saturating) |
| 0x10C | `LAST_ERR_INFO` | RO | 最近錯誤資訊 (width depends on N) |
| 0x110 | `MC_STATUS` | RO | Multicast 狀態 |
| 0x114 | `MC_FAIL_COUNT` | RO | Multicast 失敗計數 (ERR_COUNTER_WIDTH, saturating) |

### 4.2 Error Status Register (0x100)

| Field | Bit | Description |
|-------|-----|-------------|
| `ecc_uncorr_err` | [0] | ECC Uncorrectable 錯誤發生 |
| `timeout_err` | [1] | Timeout 錯誤發生 |
| `multicast_partial` | [2] | Multicast 部分失敗 |
| `multicast_fail` | [3] | Multicast 全部失敗 |
| Reserved | [7:4] | — |

### 4.3 Last Error Info Register (0x10C)

Register 寬度固定（NODE_ID_WIDTH = 8）：

| Field | Bit | Width | Description |
|-------|-----|-------|-------------|
| `err_axi_id` | [7:0] | 8 | 錯誤 transaction 的 AXI ID |
| `err_src_id` | [15:8] | 8 | 錯誤來源 node ID |
| `err_dst_id` | [23:16] | 8 | 錯誤目標 node ID |
| Reserved | [31:24] | 8 | — |

### 4.4 Error Counter Behavior

所有 error counter 使用 **saturating arithmetic**：

| Counter | Width | Saturation | Clear |
|---------|-------|------------|-------|
| `ERR_COUNT` | ERR_COUNTER_WIDTH (default 16) | 2^W - 1 | Write 1 to ERR_STATUS[0] |
| `ECC_UNCORR_ERR_CNT` | ERR_COUNTER_WIDTH (default 16) | 2^W - 1 | Write 1 to ERR_STATUS[0] |
| `MC_FAIL_COUNT` | ERR_COUNTER_WIDTH (default 16) | 2^W - 1 | Write 1 to MC_STATUS[0] |

**Saturation Behavior:**
```
if (counter < MAX_VALUE)
    counter = counter + 1;
// else: counter stays at MAX_VALUE (no wrap-around)
```

> **設計理由**：使用 saturating counter 而非 wrap-around，確保軟體讀取時永遠知道「至少發生了這麼多錯誤」。若 counter 已飽和，軟體應優先處理錯誤並清除 counter。

---

## 5. Router QoS Arbitration

Router 使用 flit header 的 `qos` 欄位進行 arbitration。

### 5.1 Arbitration Policy

```
Priority order:
1. Higher qos value wins (qos=15 highest, qos=0 lowest)
2. Same qos: Round-robin among same-priority flits
3. Wormhole: Body/Tail flits follow Head flit (no preemption)
```

### 5.2 QoS-Aware Arbitration

```systemverilog
// Simplified arbitration logic
always_comb begin
    winner = '0;
    max_qos = '0;

    for (int i = 0; i < NUM_INPUTS; i++) begin
        if (valid[i] && !blocked[i]) begin
            if (flit[i].hdr.qos > max_qos) begin
                max_qos = flit[i].hdr.qos;
                winner = i;
            end else if (flit[i].hdr.qos == max_qos) begin
                // Round-robin among same priority
                if (rr_priority[i])
                    winner = i;
            end
        end
    end
end
```

---

## 6. Design Considerations

### 6.1 QoS vs Deadlock

高 QoS traffic 不應造成低 QoS traffic 永久 starvation：

| 機制 | 說明 |
|------|------|
| Age-based promotion | 等待過久的 flit 自動提升 qos |
| Minimum bandwidth guarantee | Regulator mode 保證最低頻寬 |
| Separate VC per QoS class | 不同 QoS 使用不同 Virtual Channel |

### 6.2 Multicast QoS

Multicast transaction 的 QoS 處理：

```
Request phase:
  - Multicast request 使用發送端設定的 qos
  - 所有 target 收到相同 qos 的 request

Response aggregation:
  - 所有 target response 繼承原始 request 的 qos
  - Response aggregator 收齊所有 response 後合併
  - 合併後的 B response qos = original_request.qos
```

> **bresp 合併規則**：取 worst case（任一 SLVERR → 最終 SLVERR）

### 6.3 Error Response QoS

Response flit 的 qos **繼承自對應 request**，不因錯誤而修改：

```
response.qos = original_request.qos;  // 無論 bresp 為何
```

> **設計理由**：保持簡單一致的行為。高優先級 request 的錯誤 response 自然也是高優先級。若需要錯誤快速通知，應在 request 發送時就使用適當的 qos。

---

## 7. Hardware Integration

本節從硬體設計角度描述 QoS 機制如何與現有 NI、Router、Flow Control 整合運作。

### 7.1 系統架構總覽

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Request Path                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AXI Master          NMU                   Router Chain        NSU │
│ ┌─────────┐    ┌───────────────────┐      ┌──────────────────┐   ┌───────┐ │
│ │ AW/AR   │───▶│ QoS Generator     │─────▶│ QoS-Aware        │──▶│       │ │
│ │ awqos   │    │ (計算 flit.qos)   │      │ WormholeArbiter  │   │ Store │ │
│ │ arqos   │    │                   │      │ (優先高 qos)      │   │  qos  │ │
│ └─────────┘    └───────────────────┘      └──────────────────┘   └───────┘ │
│                         │                          │                   │    │
│                         ▼                          ▼                   ▼    │
│                 ┌──────────────┐           ┌─────────────┐     ┌──────────┐│
│                 │flit.hdr.qos │           │ Arbitration │     │ Response ││
│                 │ (4 bits)    │           │ Priority    │     │ 繼承 qos ││
│                 └──────────────┘           └─────────────┘     └──────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 QoS 計算時機與位置

#### 7.2.1 關鍵問題：誰算？什麼時候算？用什麼資料算？

| 問題 | 答案 |
|------|------|
| **誰負責計算 QoS？** | NMU 內的 **QoS Generator** 子模組 |
| **什麼時候計算？** | 產生 **AW/AR flit** 時（不是 W flit） |
| **用什麼資料計算？** | AXI 信號 + 內部狀態（視 mode 而定） |
| **W flit 的 QoS 從哪來？** | **繼承**對應 AW flit 的 qos |

#### 7.2.2 與 Flit 產生時序的關係

```
Timeline: AXI Write Transaction → Flit 產生

T0: AXI AW 到達 NMU
    ├─ 緩衝到 _aw_input_fifo
    ├─ 從 awlen + awsize 計算 transaction_bytes
    └─ 【尚未產生 flit】

T1~Tn: AXI W 資料陸續到達
    ├─ 緩衝到 _w_input_fifo
    └─ 【尚未產生 flit】

Tn+1: 最後一個 W (wlast=1) 到達
    │
    ├─ 【QoS Generator 在此時被呼叫】
    │   ├─ 輸入: awqos, transaction_bytes, 內部狀態
    │   └─ 輸出: final_qos (4 bits)
    │
    └─ PacketAssembler 產生 flit 序列:
        ├─ AW_Flit: hdr.qos = final_qos     ◄── QoS 寫入
        ├─ W_Flit[0]: hdr.qos = final_qos   ◄── 繼承
        ├─ W_Flit[1]: hdr.qos = final_qos   ◄── 繼承
        └─ ...
```

### 7.3 各 Mode 的詳細運作

#### 7.3.0 關鍵概念：為什麼不需要知道網路狀況？

每個 Node 的 QoS Generator 是**本地模組**，它完全不知道：
- 網路目前有多擁塞
- 其他 Node 在發什麼
- Router 的 buffer 有多滿

**那它們怎麼能有效調整 QoS？**

| Mode | 它知道什麼 | 它不知道什麼 | 運作原理 |
|------|-----------|-------------|----------|
| **Limiter** | 自己發了多少 bytes | 網路是否擁塞 | **開環控制**：預防性自我約束 |
| **Regulator** | Response 回來的速度 | 網路內部狀態 | **閉環控制**：從結果反推網路狀況 |

**Limiter = 開環控制（預防性）**
```
「我不管網路怎樣，我就是限制自己不要發太快。」
→ 防止單一 Master 獨佔網路資源
→ 不需要反饋，只需要知道自己發了多少
```

**Regulator = 閉環控制（反饋調節）**
```
「我從 response 回來的速度，推斷我是否獲得足夠頻寬。」
→ Response 回來慢 = 網路擁塞 = 我的頻寬不足 → 提高 QoS
→ Response 回來快 = 網路暢通 = 我的頻寬充足 → 降低 QoS
```

```
Regulator 的閉環控制：

                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
              ┌───────────┐                               │
期望頻寬 ────▶│  比較器   │──▶ 差值 ──▶ urgency ──▶ qos   │
(BUDGET)      └───────────┘                    │          │
                    ▲                          │          │
                    │                          ▼          │
              ┌───────────┐              ┌──────────┐     │
              │  實際頻寬  │◀─────────────│  網路    │◀────┘
              │(response) │              │(黑盒子)  │
              └───────────┘              └──────────┘

              雖然不知道網路內部狀態，
              但可以從「結果」(response 速度) 推斷！
```

---

#### 7.3.1 Bypass Mode（預設）

**硬體行為**：直通，無額外邏輯。

```
輸入: awqos (from AXI AW)
輸出: flit.hdr.qos = awqos

         ┌─────────────┐
awqos ──▶│   直接     │──▶ flit.hdr.qos
         │   傳遞     │
         └─────────────┘
```

**使用場景**：AXI Master 已經設定好適當的 qos 值。

---

#### 7.3.2 Fixed Mode

**硬體行為**：忽略 AXI qos，使用 CSR 配置的固定值。

```
輸入: awqos (ignored)
輸出: flit.hdr.qos = CSR.QOS_FIXED_VALUE

         ┌─────────────┐
awqos ──▶│   忽略     │
         │            │──▶ flit.hdr.qos
CSR   ──▶│ FIXED_VALUE│
         └─────────────┘
```

**使用場景**：所有來自此 NI 的 traffic 使用相同優先級。

---

#### 7.3.3 Limiter Mode

**目的**：限制單一 Master 的頻寬上限，防止獨佔網路。

**核心問題解答**：

| 問題 | 答案 |
|------|------|
| **用什麼依據計算？** | `transaction_bytes` = 這筆 AXI transaction 的總 bytes |
| **transaction_bytes 從哪來？** | 從 AW 的 `awlen` + `awsize` 計算：`bytes = (awlen + 1) × (1 << awsize)` |
| **什麼時候計算？** | 產生 AW flit 時 |
| **內部狀態是什麼？** | `bandwidth_counter`：累計的「超額」bytes |

**硬體架構**：

```
                      ┌────────────────────────────────────────────────┐
                      │              Limiter Mode                       │
                      ├────────────────────────────────────────────────┤
                      │                                                 │
 awlen, awsize ──────▶│  ┌─────────────────────────────────────────┐   │
                      │  │ transaction_bytes = (awlen+1) × 2^awsize│   │
                      │  └──────────────────┬──────────────────────┘   │
                      │                     │                          │
                      │                     ▼                          │
                      │  ┌──────────────────────────────────────────┐  │
      每個 cycle ────▶│  │        bandwidth_counter                 │  │
  (減 BANDWIDTH_LIMIT)│  │  ┌────────────────────────────────────┐  │  │
                      │  │  │ counter += transaction_bytes       │  │  │
                      │  │  │ counter -= BANDWIDTH_LIMIT (每cycle)│  │  │
                      │  │  └────────────────────────────────────┘  │  │
                      │  └──────────────────┬───────────────────────┘  │
                      │                     │                          │
                      │                     ▼                          │
                      │  ┌──────────────────────────────────────────┐  │
                      │  │ counter > SATURATION_THRESHOLD ?         │  │
                      │  │   YES → qos = LOW_PRIORITY               │  │
                      │  │   NO  → qos = CSR.QOS_FIXED_VALUE        │  │
                      │  └──────────────────┬───────────────────────┘  │
                      │                     │                          │
                      └─────────────────────┼──────────────────────────┘
                                            │
                                            ▼
                                      flit.hdr.qos
```

**數值範例**：

配置：`BANDWIDTH_LIMIT=256` (1/256 × 256 = 1 byte/cycle)，`SATURATION_THRESHOLD=1024`

| Cycle | 事件 | transaction_bytes | counter 變化 | counter | qos |
|-------|------|-------------------|--------------|---------|-----|
| 0 | - | - | 初始 | 0 | - |
| 1 | AW (256B burst) | 256 | +256 - 256 | 0 | HIGH |
| 2 | - | - | -256 | -256 | - |
| 3 | AW (1KB burst) | 1024 | +1024 - 256 | 512 | HIGH |
| 4 | - | - | -256 | 256 | - |
| 5 | AW (2KB burst) | 2048 | +2048 - 256 | 2048 | **LOW** ← 超過閾值 |
| 6 | - | - | -256 | 1792 | - |
| 7 | AW (64B burst) | 64 | +64 - 256 | 1600 | **LOW** |
| 8 | - | - | -256 | 1344 | - |
| 9 | - | - | -256 | 1088 | - |
| 10 | AW (64B burst) | 64 | +64 - 256 | 896 | HIGH ← 恢復 (< 1024) |

**解讀**：當 Master 短時間發送大量資料（超過 1 byte/cycle 平均），counter 累積，觸發低優先級，讓其他 Master 有機會使用網路。

**使用場景：DMA vs CPU 競爭**

```
情境：Node A (DMA) 和 Node B (CPU) 同時存取 Node C 的 Memory

沒有 Limiter：
  Cycle 1-10: DMA 發送 10KB (qos=8)
              大量 flit 塞滿 Router
              CPU 的 64B request (qos=8) 被擋在後面

  問題：DMA 和 CPU 的 qos 相同，先到先服務，DMA 獨佔網路

有 Limiter（DMA 啟用）：
  Cycle 1: DMA 發送 2KB → counter > threshold → qos=2 (LOW)
  Cycle 2: CPU 發送 64B → qos=8 (HIGH)
           Router Arbitration: CPU wins! (qos=8 > qos=2)

  結果：CPU 的緊急 request 可以「插隊」通過
```

---

#### 7.3.4 Regulator Mode

**目的**：保證 Master 獲得最低頻寬。當實際頻寬不足時，提高 QoS。

**核心問題解答**：

| 問題 | 答案 |
|------|------|
| **用什麼依據計算？** | `response_bytes` = 收到的 response 帶回的 bytes |
| **response_bytes 從哪來？** | NSU → NMU 的 B/R response |
| **什麼時候更新狀態？** | 每 cycle（減 budget）+ 收到 response 時（加 bytes） |
| **什麼時候計算 QoS？** | 產生 AW/AR flit 時，讀取當前 urgency_level |

**與 Limiter 的關鍵差異**：

| 特性 | Limiter | Regulator |
|------|---------|-----------|
| **追蹤對象** | 發送的 request bytes | 收到的 response bytes |
| **目的** | 限制上限 | 保證下限 |
| **調整方向** | 超標時降低 | 不足時提升 |
| **控制類型** | 開環控制（預防性） | 閉環控制（反饋調節） |
| **counter 語義** | 正值 = 超額使用 | 負值 = 頻寬不足 |

> **設計注意**：雖然兩者都使用 `bandwidth_counter`，但語義不同：
> - Limiter：counter 累加「發送量」，正值表示「超出限制」
> - Regulator：counter 累加「收到量」減「預期量」，負值表示「不足」
>
> 硬體實作時，兩者應視為獨立的邏輯模組，根據 QOS_MODE 選擇使用。

**Regulator 跨路徑資訊傳遞**：

Regulator 的 `response_bytes` 來自 RspPath，但 QoS 計算在 ReqPath：

```
NMU 內部：

┌─────────────────────────────────────────────────────────────────┐
│  RspPath                           ReqPath                       │
│  ┌──────────────────┐              ┌──────────────────┐          │
│  │ process_cycle(): │              │ QoS Generator:   │          │
│  │   if (B/R 收到)  │──────────────│   bandwidth_     │          │
│  │     notify(bytes)│  內部信號    │   counter        │          │
│  └──────────────────┘              │   urgency_level  │          │
│                                    └──────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

**硬體架構**：

```
                      ┌─────────────────────────────────────────────────┐
                      │               Regulator Mode                     │
                      ├─────────────────────────────────────────────────┤
                      │                                                  │
                      │  ┌───────────────────────────────────────────┐  │
  response_bytes ────▶│  │          bandwidth_counter                │  │
  (從 B/R response)   │  │  ┌─────────────────────────────────────┐  │  │
                      │  │  │ counter += response_bytes           │  │  │
      每個 cycle ────▶│  │  │ counter -= BANDWIDTH_BUDGET (每cycle)│  │  │
  (減 BANDWIDTH_BUDGET)│ │  └─────────────────────────────────────┘  │  │
                      │  └─────────────────┬─────────────────────────┘  │
                      │                    │                            │
                      │                    ▼                            │
                      │  ┌───────────────────────────────────────────┐  │
                      │  │          urgency_level                    │  │
                      │  │  ┌─────────────────────────────────────┐  │  │
                      │  │  │ if counter < 0: urgency++           │  │  │
                      │  │  │ if counter > 0: urgency--           │  │  │
                      │  │  │ clamp to [0, MAX_URGENCY]           │  │  │
                      │  │  └─────────────────────────────────────┘  │  │
                      │  └─────────────────┬─────────────────────────┘  │
                      │                    │                            │
                      │                    ▼  (產生 AW/AR flit 時讀取)  │
                      │  ┌───────────────────────────────────────────┐  │
                      │  │ raw_qos = BASE_QOS + urgency_level        │  │
                      │  │ clamped_qos = min(raw_qos, 15)            │  │
                      │  │ final_qos = max(clamped_qos, SOCKET_QOS)  │  │
                      │  └─────────────────┬─────────────────────────┘  │
                      │                    │                            │
                      └────────────────────┼────────────────────────────┘
                                           │
                                           ▼
                                     flit.hdr.qos
```

**完整時序圖**：

```
NMU                                              NSU
   │                                                    │
   │  Cycle 1: 發送 Request (qos=BASE_QOS+urgency)     │
   │────────────────────────────────────────────────────▶
   │                                                    │
   │  Cycle 2-10: 等待處理...                          │
   │  urgency_level 持續根據 counter 調整               │
   │  (每 cycle: counter -= BANDWIDTH_BUDGET)          │
   │  (counter < 0 → urgency++)                        │
   │                                                    │
   │  Cycle 11: 收到 Response (256 bytes)              │
   │◀────────────────────────────────────────────────────
   │  counter += 256                                   │
   │  (counter > 0 → urgency--)                        │
   │                                                    │
   │  Cycle 12: 發送下一筆 Request                     │
   │  qos = BASE_QOS + (新的 urgency_level)            │
   │────────────────────────────────────────────────────▶
```

**使用場景：Video Decoder 保證頻寬**

```
情境：Node A (Video Decoder) 需要穩定 100 bytes/cycle 的頻寬
配置：BANDWIDTH_BUDGET=100, BASE_QOS=4, URGENCY_WIDTH=3 (max=7)

情境 1：網路暢通
  Cycle 1-5: 發送 Request，Response 快速回來
             counter ≈ 0（實際頻寬 ≈ 目標頻寬）
             urgency = 2~3
             qos = 4 + 2 = 6（穩定）

情境 2：網路擁塞（其他 Node 也在大量傳輸）
  Cycle 1-5: 發送 Request，Response 遲遲不回來
             counter = 0 - 100×5 = -500（頻寬嚴重不足）
             urgency = 0 → 1 → 2 → 3 → 4 → 5
             qos = 4 + 5 = 9（自動提高！）

  Cycle 6-10: urgency 繼續上升
              qos = 4 + 7 = 11（接近最高）

              此時 Video Decoder 的 flit 在 Router 競爭時會贏！
              → 開始收到更多 Response
              → counter 回升，urgency 下降
              → 系統自動平衡

結果：即使網路擁塞，Video Decoder 也能透過提高 QoS 來爭取頻寬
```

---

### 7.4 Router 如何使用 QoS

#### 7.4.1 QoS-Aware Arbitration

**發生時機**：Mesh cycle 的 **Phase 4 (Route and Forward)**

**原有行為**（無 QoS）：
```
多個 input 競爭同一 output → Round-Robin 選擇
```

**新行為**（有 QoS）：
```
多個 input 競爭同一 output
  → Step 1: 找出最高 qos 值
  → Step 2: 只在最高 qos 的 inputs 之間 Round-Robin
```

**硬體邏輯**：

```
             Input 0 (qos=5)  ─┐
             Input 1 (qos=12) ─┼──▶ ┌─────────────────────────┐
             Input 2 (qos=3)  ─┤    │   QoS-Aware Arbiter     │
             Input 3 (qos=12) ─┘    │                         │
                                    │  1. max_qos = 12        │
                                    │  2. candidates = {1, 3} │
                                    │  3. RR among {1, 3}     │
                                    │                         │
                                    └───────────┬─────────────┘
                                                │
                                                ▼
                                         Winner: Input 1 或 3
                                         (輪流，公平)
```

#### 7.4.2 Wormhole Switching 與 QoS 的交互

**重要**：QoS arbitration 只發生在 **HEAD flit**。

```
Packet A (qos=15): [HEAD] → [BODY] → [BODY] → [TAIL]
Packet B (qos=3):  [HEAD] → [BODY] → [TAIL]

Cycle 1: A.HEAD vs B.HEAD 競爭 output
         A wins (qos=15 > qos=3)
         A.HEAD 獲得 path lock

Cycle 2: A.BODY 自動通過（已鎖定，不需重新 arbitrate）
         B.HEAD 仍在等待

Cycle 3: A.BODY 通過

Cycle 4: A.TAIL 通過，釋放 lock

Cycle 5: B.HEAD 獲得 output（此時沒有競爭）
```

**設計理由**：
- Wormhole switching 要求 packet 不可被中斷
- 若允許 body/tail 被 preempt，會造成 deadlock
- 因此 QoS 只影響 HEAD flit 的競爭，不影響已鎖定的 path

**Priority Inversion 問題與處理**：

```
情境：低 QoS 的長 packet 阻擋高 QoS 的短 packet

Cycle 1: Low QoS Packet A (10 flits) 的 HEAD 獲得 path lock
Cycle 2: High QoS Packet B (2 flits) 的 HEAD 到達
         → B 無法 preempt A（Wormhole 特性）
         → B 必須等待 A 的全部 10 flits 傳完

這是 Priority Inversion！
```

**本設計的處理策略**：**接受 Priority Inversion，依賴其他機制緩解**

| 緩解機制 | 說明 | 本設計狀態 |
|----------|------|------------|
| **短 packet 設計** | 限制 packet 最大長度，減少 blocking 時間 | AXI burst length 限制 |
| **Age-based promotion** | 等待過久的 flit 自動提升 qos | 可選實作（Section 6.1） |
| **Per-QoS Virtual Channel** | 不同 QoS 使用不同 VC，物理隔離 | 未來擴展 |

> **設計決策**：本設計選擇「簡單的 Wormhole + QoS」而非「完全的 QoS 隔離」，因為：
> 1. 硬體複雜度較低
> 2. 對大多數應用場景已足夠
> 3. 若需要嚴格的 QoS 保證，應使用 Per-QoS VC（需要額外的 buffer 和 arbitration 邏輯）

---

### 7.5 Response Path 的 QoS 處理

#### 7.5.1 繼承機制

Response flit 的 qos **必須繼承**對應 request 的 qos：

```
NSU 內部：

┌─────────────────────────────────────────────────────────┐
│                Transaction Table                         │
├──────────┬──────────┬──────────┬──────────┬────────────┤
│ rob_idx  │ axi_id   │ src_id   │ qos      │ ...        │
├──────────┼──────────┼──────────┼──────────┼────────────┤
│ 0        │ 5        │ (3,2)    │ 12       │            │
│ 1        │ 3        │ (1,0)    │ 7        │            │
│ ...      │          │          │          │            │
└──────────┴──────────┴──────────┴──────────┴────────────┘

收到 Request (rob_idx=0, qos=12):
  → 存入 table[0].qos = 12

產生 Response (rob_idx=0):
  → 讀取 table[0].qos = 12
  → response_flit.hdr.qos = 12
```

#### 7.5.2 為什麼 Response 要繼承 QoS？

| 理由 | 說明 |
|------|------|
| **優先級一致性** | 高優先級 request 的 response 自然也應該優先 |
| **端到端延遲** | 若 response 被降級，會拖累整體 transaction 延遲 |
| **簡化設計** | Response path 不需要獨立的 QoS Generator |

---

### 7.6 QoS 與 Flow Control 的交互

#### 7.6.1 Credit-Based Flow Control 不受 QoS 影響

**重要**：QoS 只影響 **arbitration**，不影響 **flow control**。

```
情境：高 qos flit 要發送，但下游 buffer 滿了

Router A                          Router B
   │                                 │
   │  flit (qos=15)                  │
   │  credit = 0 (B 滿了)            │
   │  out_ready = 0                  │
   │                                 │
   │  ✗ 無法發送                     │
   │  (即使 qos 最高)                │
   │                                 │
   │  ... 等待 B 釋放空間 ...        │
   │                                 │
   │  credit = 1                     │
   │  out_ready = 1                  │
   │                                 │
   │  ✓ 現在可以發送                 │
   │─────────────────────────────────▶
```

**設計理由**：Flow control 是為了防止 buffer overflow，這是比 QoS 更基本的正確性要求。

#### 7.6.2 QoS 不會造成 Starvation（設計建議）

為防止低 qos traffic 永久飢餓，建議加入 **Age-based Promotion**：

```
每個 flit 有 age counter:
  - 每 cycle 在 buffer 中等待 → age++
  - age > AGE_THRESHOLD → 提升 effective_qos

effective_qos = min(flit.hdr.qos + (age / AGE_STEP), 15)
```

---

### 7.7 Summary: 完整資料流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ AXI Master 發起 Write Transaction                                            │
│                                                                              │
│  AW: awaddr=0x1000, awlen=3, awsize=5, awqos=8                              │
│  W[0-3]: wdata...                                                           │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ NMU: 收到 AW + W，準備產生 Flit                                          │
│                                                                              │
│  1. 計算 transaction_bytes = (3+1) × 32 = 128 bytes                         │
│  2. 呼叫 QoS Generator:                                                      │
│     - Bypass: final_qos = 8 (直接使用 awqos)                                │
│     - Fixed:  final_qos = CSR.FIXED_VALUE                                   │
│     - Limiter: counter += 128; final_qos = (counter > threshold) ? LOW : 8  │
│     - Regulator: final_qos = BASE_QOS + urgency_level                       │
│  3. 產生 Flit:                                                               │
│     - AW_Flit (hdr.qos = final_qos)                                         │
│     - W_Flit × 4 (hdr.qos = final_qos, 繼承)                                │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Router Chain: Flit 經過多個 Router                                           │
│                                                                              │
│  每個 Router, 每個 Cycle:                                                    │
│    Phase 4: Arbitration                                                      │
│      - 若多個 input 競爭同一 output                                          │
│      - 選擇 qos 最高的 (同 qos 則 Round-Robin)                               │
│      - HEAD flit 獲得 path lock，後續 BODY/TAIL 自動通過                     │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ NSU: 收到 Flit，執行 Memory 操作                                        │
│                                                                              │
│  1. 存入 Transaction Table: {rob_idx, axi_id, src_id, qos=final_qos}        │
│  2. 重組為 AXI AW + W，發送到 Memory                                         │
│  3. 收到 AXI B Response                                                      │
│  4. 產生 B_Flit，qos = table[rob_idx].qos (繼承)                             │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Response 返回 NMU                                                        │
│                                                                              │
│  1. B_Flit 經由 Response Router 返回 (使用繼承的 qos)                        │
│  2. NMU 收到 B_Flit                                                      │
│  3. 若 Regulator Mode: counter += response_bytes，更新 urgency_level        │
│  4. 產生 AXI B Response 給 Master                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Related Documents

- [Flit Format](02_flit.md) - Header qos 欄位定義、NODE_ID_WIDTH、QOS_WIDTH
- [Network Interface](04_network_interface.md) - NI 架構
- [Router](03_router.md) - Router arbitration

### Design References

本設計的 QoS Generator（Fixed/Limiter/Regulator modes）、Performance Probe（Packet/Transaction）、flit header qos field（4 bits）等機制，綜合參考業界主流 NoC QoS 架構設計而成。
