---
document_id: NOC-SPEC-06
title: QoS and Performance Monitoring
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [02_flit.md, 04_network_interface.md]
---

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
| **Bypass** | 直接使用 AXI awqos/arqos，不做任何調整 | 預設行為，AXI Master 自行管理 QoS |
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

Arbitration 在所有 valid 且未 blocked 的 input 中，選出 `qos` 最高者。同 `qos` 時以 Round-Robin 輪替。Wormhole body/tail flit 跟隨 head flit 的 path lock，不參與重新 arbitration。

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

## 7. Integration Notes

### 7.1 QoS 計算時機

| 問題 | 規則 |
|------|------|
| 計算位置 | NMU 內的 QoS Generator |
| 計算時機 | 產生 AW/AR flit 時 |
| W flit QoS | 繼承對應 AW flit 的 qos |
| Response QoS | NSU 從 Transaction Table 讀取原始 request qos，直接繼承 |

### 7.2 Limiter vs Regulator 控制模型

| 特性 | Limiter | Regulator |
|------|---------|-----------|
| 控制類型 | 開環（預防性） | 閉環（反饋調節） |
| 追蹤對象 | 發送的 request bytes | 收到的 response bytes |
| counter 語義 | 正值 = 超額使用 | 負值 = 頻寬不足 |
| 調整方向 | 超標時降低 QoS | 不足時提升 QoS |

> Regulator 的 `response_bytes` 來自 RspPath（B/R response），但 QoS 計算發生在 ReqPath。NMU 內部透過信號將 response 資訊傳遞給 QoS Generator。

### 7.3 Wormhole 與 QoS 交互

- QoS arbitration **僅在 HEAD flit** 發生，body/tail 跟隨已鎖定的 path
- 高 QoS 無法 preempt 已鎖定的低 QoS wormhole path（Priority Inversion）
- 緩解方式：AXI burst length 限制（縮短 blocking 時間）、Age-based promotion（可選）、Per-QoS VC（未來擴展）

### 7.4 QoS 與 Flow Control

QoS 只影響 **arbitration**，不影響 **flow control**。即使 qos=15，若下游 credit=0 或 ready=0，仍無法發送。Flow control 的正確性優先於 QoS 優先級。

---

## Related Documents

- [Flit Format](02_flit.md) - Header qos 欄位定義、NODE_ID_WIDTH、QOS_WIDTH
- [Network Interface](04_network_interface.md) - NI 架構
- [Router](03_router.md) - Router arbitration

### Design References

本設計的 QoS Generator（Fixed/Limiter/Regulator modes）、Performance Probe（Packet/Transaction）、flit header qos field（4 bits）等機制，綜合參考業界主流 NoC QoS 架構設計而成。

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
