# Width Converter

本文件描述 Width Converter 元件，用於橋接不同 AXI bus 寬度與 NoC 固定 408-bit flit 格式之間的差異。所有參數依據 [Flit Format](02_flit.md) 之固定參數設計。

---

## 1. 設計動機

### 1.1 問題描述

NoC 採用固定 408-bit flit 格式，其中 W/R channel 的 payload 攜帶 256-bit data（`AXI_DATA_WIDTH = 256`）。然而系統中不同節點可能使用不同的 AXI bus 寬度：

| 節點類型 | 典型 AXI Data Width |
|----------|---------------------|
| Peripheral controller | 32-bit |
| Sensor interface | 64-bit |
| CPU | 128-bit |
| DMA Engine | 256-bit（原生） |
| High-bandwidth accelerator | 512-bit |
| GPU / HBM controller | 1024-bit |

當本地 AXI 寬度不等於 256-bit 時，需要 Width Converter 執行寬度轉換，使 NI 始終以 256-bit AXI 介面運作。

### 1.2 設計目標

1. 支援 AXI spec 定義的資料寬度：32b、64b、128b、256b、512b、1024b
2. 透明轉換 — 不改變 AXI protocol 語意
3. 維持 AXI ordering 與 transaction 語意
4. 正確處理 wstrb 轉換與 byte lane 對應
5. 當 burst length 因 downsizing 超過 255 時，支援 transaction splitting
6. 正確合併 split transaction 的 response（worst-case bresp/rresp）

---

## 2. 系統位置

Width Converter 放置於 **本地 AXI 介面與 NI 之間**（NI packetization 之前），確保 NI 始終看到 256-bit AXI 介面：

```
Local AXI (Nb)  ←→  Width Converter  ←→  NI (256b AXI)  ←→  Router (408b flit)
                    (Width Conv,          (RoB, Pack/Unpack,
                     Split, Merge)         ECC, Header)
```

**元件職責：**

| 元件 | 職責 |
|------|------|
| **Width Converter** | AXI 寬度轉換、burst parameter 調整、transaction splitting、response merging |
| **NI** | Flit packetization/depacketization、RoB、ECC generate/check、header 填充 |
| **Router** | 408-bit flit 轉發、XY routing、arbitration |

當本地 AXI 寬度為 256-bit 時，Width Converter 為 bypass 模式（wire-through），無轉換開銷。

---

## 3. 轉換場景

### 3.1 Narrow AXI（< 256b）— Upsizing

本地 AXI 寬度小於 256-bit 時，需將多個 AXI beats 打包成一個 256-bit data beat 後送入 NI：

```
Local AXI (64b)                    NI 側 (256b)
┌──────────────┐                  ┌──────────────────────────────┐
│ Beat 0: 64b  │──┐              │                              │
├──────────────┤  │              │   256-bit data               │
│ Beat 1: 64b  │──┼─── pack ───►│   = {beat3, beat2,           │
├──────────────┤  │              │      beat1, beat0}           │
│ Beat 2: 64b  │──┤              │                              │
├──────────────┤  │              │   wstrb = 聚合 4 beats       │
│ Beat 3: 64b  │──┘              └──────────────────────────────┘

4 Local beats → 1 NI beat
```

### 3.2 Native AXI（256b）— Bypass

本地 AXI 寬度等於 256-bit 時，直接透傳，無轉換：

```
Local AXI (256b)                   NI 側 (256b)
┌──────────────────┐              ┌──────────────────┐
│ 256-bit data     │── bypass ──►│ 256-bit data     │
└──────────────────┘              └──────────────────┘

1:1 wire-through
```

### 3.3 Wide AXI（> 256b）— Downsizing

本地 AXI 寬度大於 256-bit 時，需將一個寬 AXI beat 拆分為多個 256-bit data beats 送入 NI：

```
Local AXI (1024b)                  NI 側 (256b each)
┌──────────────────────────┐      ┌──────────────────┐
│                          │      │ data[255:0]      │  NI Beat 0
│  1024-bit data           │      ├──────────────────┤
│                          │      │ data[511:256]    │  NI Beat 1
│  wstrb: 128 bytes        │─────►├──────────────────┤
│                          │      │ data[767:512]    │  NI Beat 2
│                          │      ├──────────────────┤
│                          │      │ data[1023:768]   │  NI Beat 3
└──────────────────────────┘      └──────────────────┘

1 Local beat → 4 NI beats
```

---

## 4. Write Path（AW, W, B Channels）

### 4.1 AW Channel 處理

#### 4.1.1 Upsizing（窄 → 256b）

當本地 AXI 寬度 < 256b 時，Width Converter 調整 burst parameters 使 NI 看到 256b 語意：

- **awsize**：提升至 `3'b101`（32 bytes = 256 bits）
- **awlen**：重新計算 — 打包後的 beat 數減少
- **awaddr**：對齊至 256-bit（32-byte）邊界

```
計算公式：
  ratio       = 256 / AXI_DATA_WIDTH
  aligned_len = ceil((original_len + 1 + start_offset) / ratio) - 1
  new_addr    = awaddr & ~(32 - 1)   // 對齊至 32-byte 邊界
  start_offset = (awaddr % 32) / (AXI_DATA_WIDTH / 8)
```

**範例：** 64b AXI → 256b NI
- 原始：awaddr = 0x108, awlen = 15, awsize = 3'b011（8 bytes）
- ratio = 256 / 64 = 4
- start_offset = (0x108 % 32) / 8 = 1
- aligned_len = ceil((16 + 1) / 4) - 1 = 4
- 轉換後：awaddr = 0x100, awlen = 4, awsize = 3'b101

#### 4.1.2 Downsizing（寬 → 256b）

當本地 AXI 寬度 > 256b 時，可能需要 **transaction splitting**：

1. **計算新的 burst length：**
   ```
   ratio   = AXI_DATA_WIDTH / 256
   new_len = (original_len + 1) * ratio - 1
   ```

2. **檢查是否需要 split：**
   ```
   if new_len > 255:
       將 transaction 切分為多個 sub-transactions
   ```

3. **Split transaction 處理：**
   - 第一個 sub-transaction：原始 address，awlen = 255
   - 後續 sub-transaction：調整後的 address，剩餘 beats

**範例：** 1024b AXI → 256b NI（ratio = 4）
- 原始：awaddr = 0x0000, awlen = 127（128 beats x 128B = 16 KB）
- new_len = 128 x 4 - 1 = 511（> 255，需要 split）
- Sub-txn 0：awaddr = 0x0000, awlen = 255（256 beats x 32B = 8 KB）
- Sub-txn 1：awaddr = 0x2000, awlen = 255（256 beats x 32B = 8 KB）

#### 4.1.3 AW Channel State Machine

```
                    ┌─────────────────┐
                    │      IDLE       │
                    └────────┬────────┘
                             │ aw_valid
                             ▼
                    ┌─────────────────┐
              ┌─────│  CALC_PARAMS    │─────┐
              │     └─────────────────┘     │
              │                             │
        no_split                      needs_split
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │  PASSTHROUGH    │           │  SPLIT_FIRST    │
    └────────┬────────┘           └────────┬────────┘
             │ ni_aw_ready                 │ ni_aw_ready
             │                             ▼
             │                    ┌─────────────────┐
             │               ┌──►│  SPLIT_REMAIN   │──┐
             │               │   └────────┬────────┘  │
             │               │            │            │
             │               │     more_splits         │
             │               │            │            │
             │               └────────────┘            │
             │                            │ last_split │
             ▼                            ▼            │
       ┌─────────────┐            ┌─────────────┐     │
       │    DONE     │            │    DONE     │◄────┘
       └─────────────┘            └─────────────┘
```

### 4.2 W Channel 處理

#### 4.2.1 Upsizing（窄 → 256b）— Data Packing

收集多個窄 AXI W beats，打包成一個 256-bit data beat：

```
Local AXI (64b each)                NI 側 (256b)
┌─────────────────┐                ┌─────────────────────────────────────┐
│ wdata[63:0]     │──┐             │ wdata[255:0]                        │
│ wstrb[7:0]      │  │             │   = {local_beat3.wdata,             │
├─────────────────┤  │             │      local_beat2.wdata,             │
│ wdata[63:0]     │──┼── pack ───►│      local_beat1.wdata,             │
│ wstrb[7:0]      │  │             │      local_beat0.wdata}             │
├─────────────────┤  │             │ wstrb[31:0]                         │
│ wdata[63:0]     │──┤             │   = {local_beat3.wstrb,             │
│ wstrb[7:0]      │  │             │      ..., local_beat0.wstrb}        │
├─────────────────┤  │             └─────────────────────────────────────┘
│ wdata[63:0]     │──┘
│ wstrb[7:0]      │
└─────────────────┘
```

**wstrb 處理：**
- 每個窄 beat 的 wstrb 映射至 256-bit data 中對應的 byte lanes
- 未填充的 byte lanes 其 wstrb = 0
- 最後一組打包中若窄 beats 不足 ratio 個，padding bytes 的 wstrb = 0

#### 4.2.2 Downsizing（寬 → 256b）— Data Splitting

將一個寬 AXI W beat 拆分為多個 256-bit data beats：

```
Local AXI (512b)                    NI 側 (256b each)
┌───────────────────────┐          ┌──────────────────────┐
│ wdata[511:0]          │          │ wdata = src[255:0]   │  NI Beat 0
│ wstrb[63:0]           │── split ─│ wstrb = src[31:0]    │
│                       │    ──►  ├──────────────────────┤
│                       │          │ wdata = src[511:256] │  NI Beat 1
│                       │          │ wstrb = src[63:32]   │
└───────────────────────┘          └──────────────────────┘
```

**wstrb 切片：**
- 第 i 個 NI beat：`wstrb_ni = original_wstrb[(i+1)*32-1 : i*32]`
- 第 i 個 NI beat：`wdata_ni = original_wdata[(i+1)*256-1 : i*256]`

#### 4.2.3 W Channel Pack State Machine（Upsizing）

```
         ┌─────────────┐
         │    IDLE     │
         └──────┬──────┘
                │ local_w_valid
                ▼
         ┌─────────────┐
    ┌───►│   COLLECT   │──── 累積 local beats 至 pack buffer
    │    └──────┬──────┘
    │           │ local_w_valid && local_w_ready
    │           │
    │    beat_cnt < ratio - 1 && !local_wlast
    │           │
    └───────────┘
                │ beat_cnt == ratio - 1 || local_wlast
                ▼
         ┌─────────────┐
         │   OUTPUT    │──── 輸出打包後的 256b data 至 NI
         └──────┬──────┘
                │ ni_w_ready
                ▼
         ┌─────────────┐
         │    IDLE     │
         └─────────────┘
```

#### 4.2.4 W Channel Serialize State Machine（Downsizing）

```
         ┌─────────────┐
         │    IDLE     │
         └──────┬──────┘
                │ local_w_valid
                ▼
         ┌─────────────┐
         │   LATCH     │──── 儲存完整寬 W beat
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
    ┌───►│  SERIALIZE  │──── 輸出 data_slice[beat_cnt]
    │    └──────┬──────┘
    │           │ ni_w_ready
    │           │
    │    beat_cnt < ratio - 1
    │           │
    └───────────┘
                │ beat_cnt == ratio - 1
                ▼
         ┌─────────────┐
         │    IDLE     │
         └─────────────┘
```

### 4.3 B Channel 處理 — Response Merging

當 AW transaction 被 split 為多個 sub-transactions 時，需要合併對應的 B responses：

1. **追蹤 split 數量** — 記錄每個原始 transaction 產生了多少 sub-transactions
2. **收集 responses** — 等待所有 sub-transaction 的 B response 到達
3. **選擇最差 response** — 使用 worst-case precedence
4. **輸出單一合併 response** — 回傳給本地 AXI master

**Response Precedence（AXI spec）：**

```
DECERR (2'b11) > SLVERR (2'b10) > EXOKAY (2'b01) > OKAY (2'b00)
```

#### 4.3.1 B Channel State Machine

```
         ┌─────────────┐
         │    IDLE     │
         └──────┬──────┘
                │ ni_b_valid
                ▼
         ┌─────────────┐
         │ CHECK_SPLIT │──── 查詢是否為 split transaction
         └──────┬──────┘
                │
       ┌────────┴────────┐
       │                 │
   not_split         is_split
       │                 │
       ▼                 ▼
┌─────────────┐   ┌─────────────┐
│ PASSTHROUGH │   │   MERGE     │◄───┐
└──────┬──────┘   └──────┬──────┘    │
       │                 │           │
       │          pending_cnt > 1    │
       │                 │           │
       │                 └───────────┘
       │                 │ pending_cnt == 1
       │                 ▼
       │          ┌─────────────┐
       └─────────►│   OUTPUT    │──── local_b_valid + merged bresp
                  └─────────────┘
```

---

## 5. Read Path（AR, R Channels）

### 5.1 AR Channel 處理

AR channel 的處理與 AW channel 對稱，使用相同的 burst parameter 調整與 splitting 邏輯：

#### 5.1.1 Upsizing（窄 → 256b）

- 提升 arsize 至 `3'b101`（32 bytes）
- 重新計算 arlen（打包後 beat 數減少）
- 對齊 araddr 至 32-byte 邊界
- 記錄 start_offset 以供 R channel lane steering 使用

#### 5.1.2 Downsizing（寬 → 256b）

- 計算 new_len = (original_len + 1) x ratio - 1
- 若 new_len > 255 則 split transaction
- 追蹤 split 資訊以正確重組 R data

#### 5.1.3 AR Channel State Machine

與 AW channel state machine 相同（見 Section 4.1.3），僅訊號名稱以 `ar` 取代 `aw`。

### 5.2 R Channel 處理

#### 5.2.1 Upsizing（窄 → 256b）— Lane Steering

從 NI 接收 256-bit R data，依據地址對齊資訊提取對應 lanes 輸出至窄本地 AXI：

```
NI 側 (256b)                       Local AXI (64b each)
┌─────────────────────────────┐   ┌──────────────────────┐
│                             │   │ rdata = data[63:0]   │  Lane 0
│      256-bit rdata          │   ├──────────────────────┤
│                             │──►│ rdata = data[127:64] │  Lane 1
│  包含 4 x 64b data          │   ├──────────────────────┤
│                             │   │ rdata = data[191:128]│  Lane 2
│                             │   ├──────────────────────┤
│                             │   │ rdata = data[255:192]│  Lane 3
└─────────────────────────────┘   └──────────────────────┘

1 NI beat → 4 Local beats（lane steering）
```

#### 5.2.2 Downsizing（寬 → 256b）— Data Reassembly

從 NI 收集多個 256-bit R beats，重組成一個寬本地 AXI R beat：

```
NI 側 (256b each)                   Local AXI (1024b)
┌──────────────────────┐           ┌──────────────────────────┐
│ NI Beat 0: rdata     │──┐       │                          │
├──────────────────────┤  │       │  重組的 1024b rdata      │
│ NI Beat 1: rdata     │──┼──────►│                          │
├──────────────────────┤  │       │  = {beat3, beat2,        │
│ NI Beat 2: rdata     │──┤       │     beat1, beat0}        │
├──────────────────────┤  │       │                          │
│ NI Beat 3: rdata     │──┘       └──────────────────────────┘

4 NI beats → 1 Local beat（reassembly）
```

#### 5.2.3 R Channel Lane Steer State Machine（Upsizing）

```
         ┌─────────────┐
         │    IDLE     │
         └──────┬──────┘
                │ ni_r_valid
                ▼
         ┌─────────────┐
         │   LATCH     │──── 儲存完整 256b R beat
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
    ┌───►│ LANE_STEER  │──── 輸出 rdata[lane_idx]
    │    └──────┬──────┘
    │           │ local_r_ready
    │           │
    │    lane_idx < ratio - 1
    │           │
    └───────────┘
                │ lane_idx == ratio - 1
                ▼
         ┌─────────────┐
         │    IDLE     │
         └─────────────┘
```

#### 5.2.4 R Channel Reassembly State Machine（Downsizing）

```
         ┌─────────────┐
         │    IDLE     │
         └──────┬──────┘
                │ ni_r_valid
                ▼
         ┌─────────────┐
    ┌───►│   COLLECT   │──── 累積 NI R beats 至 reassembly buffer
    │    └──────┬──────┘
    │           │ ni_r_valid && ni_r_ready
    │           │
    │    beat_cnt < ratio - 1
    │           │
    └───────────┘
                │ beat_cnt == ratio - 1
                ▼
         ┌─────────────┐
         │   OUTPUT    │──── local_r_valid + 重組後的寬 rdata
         └──────┬──────┘
                │ local_r_ready
                ▼
         ┌─────────────┐
         │    IDLE     │
         └─────────────┘
```

### 5.3 R Response Merging

與 B channel 類似，split transaction 的 R response 必須合併：

1. **從 AR 處理繼承 split 資訊**
2. **收集 R beats** — 來自所有 sub-transactions，依序重組
3. **合併 rresp** — 使用 worst-case response precedence
4. **維護 rlast** — 僅在合併後 transaction 的最後一個 beat 設定

---

## 6. Transaction Splitting 細節

### 6.1 何時需要 Splitting

僅在 downsizing 場景（寬 AXI → 256b NI）且 burst length overflow 時需要 splitting：

```
條件：(original_len + 1) x ratio > 256
即：  new_len = (original_len + 1) x ratio - 1 > 255
```

### 6.2 Splitting 範例

| 場景 | 原始 Beats | Ratio | 轉換後 Beats | 需要 Split |
|------|-----------|-------|-------------|-----------|
| 512b → 256b, len=127 | 128 | 2 | 256 | 否（剛好 256） |
| 512b → 256b, len=255 | 256 | 2 | 512 | 是（2 sub-txns） |
| 1024b → 256b, len=63 | 64 | 4 | 256 | 否（剛好 256） |
| 1024b → 256b, len=64 | 65 | 4 | 260 | 是（2 sub-txns） |
| 1024b → 256b, len=127 | 128 | 4 | 512 | 是（2 sub-txns） |
| 1024b → 256b, len=255 | 256 | 4 | 1024 | 是（4 sub-txns） |

### 6.3 Split 追蹤結構

每個 split transaction 需要記錄追蹤資訊以正確合併 response：

```cpp
struct SplitInfo {
    uint8_t  txn_id;           // 原始 AXI transaction ID
    uint16_t total_splits;     // sub-transaction 總數
    uint16_t pending_splits;   // 尚未完成的 sub-transaction 數
    uint8_t  merged_resp;      // 累積的 worst-case response

    // Read path 專用
    uint16_t collected_beats;  // 已收集的 R beats 數量
    uint16_t expected_beats;   // 預期的 R beats 總數
};
```

### 6.4 Split Address 計算

每個 sub-transaction 的起始地址依序遞增：

```
sub_txn[0].addr = original_addr
sub_txn[i].addr = original_addr + i * 256 * 32    // 256 beats x 32 bytes
sub_txn[i].len  = min(remaining_beats, 256) - 1
```

### 6.5 Response Merging 邏輯

```cpp
// worst-case response precedence
uint8_t resp_merge(uint8_t resp_a, uint8_t resp_b) {
    // DECERR(3) > SLVERR(2) > EXOKAY(1) > OKAY(0)
    return (resp_a > resp_b) ? resp_a : resp_b;
}
```

對於 B channel：所有 sub-transaction 的 bresp 以 worst-case 合併後，回傳單一 B response 給本地 AXI master。

對於 R channel：各 sub-transaction 的 rdata 依序重組，rresp 以 worst-case 合併。rlast 僅在合併後 transaction 的最後一個 beat 設定。

---

## 7. Width Conversion Ratio 表

以下表格基於 408-bit flit 格式（56-bit header + 352-bit payload，其中 W/R data 為 256 bits）：

| Local AXI Width | Ratio | 轉換類型 | W beats 轉換 | Burst len=15 轉換後 | Max Burst 不 Split |
|-----------------|-------|----------|-------------|--------------------|--------------------|
| 32b (4B) | 8:1 | Upsizing | 8 local → 1 NI | 16 → 2 NI beats | 不適用 |
| 64b (8B) | 4:1 | Upsizing | 4 local → 1 NI | 16 → 4 NI beats | 不適用 |
| 128b (16B) | 2:1 | Upsizing | 2 local → 1 NI | 16 → 8 NI beats | 不適用 |
| 256b (32B) | 1:1 | **Bypass** | 直通 | 16 → 16 NI beats | 不適用 |
| 512b (64B) | 1:2 | Downsizing | 1 local → 2 NI | 16 → 32 NI beats | len ≤ 127 |
| 1024b (128B) | 1:4 | Downsizing | 1 local → 4 NI | 16 → 64 NI beats | len ≤ 63 |

**附註：**
- Upsizing 不會導致 burst length overflow，因此不需要 transaction splitting
- Downsizing 的 "Max Burst 不 Split" 欄位表示 original_len 的最大值，使得 `(len+1) x ratio ≤ 256`
- 超過此長度的 burst 將被 split 為多個 sub-transactions

### 7.1 Throughput 分析

| Local AXI Width | 有效資料速率 | 瓶頸端 |
|-----------------|-------------|--------|
| 32b | 32b / local cycle | Local AXI 側（窄 bus） |
| 64b | 64b / local cycle | Local AXI 側 |
| 128b | 128b / local cycle | Local AXI 側 |
| 256b | 256b / cycle | 平衡（原生寬度） |
| 512b | 256b / NI cycle | NI 側（flit 寬度限制） |
| 1024b | 256b / NI cycle | NI 側（flit 寬度限制） |

---

## 8. State Machine 總覽

### 8.1 各通道 State Machine 彙整

| Channel | Upsizing（窄 → 256b） | Downsizing（寬 → 256b） | Bypass |
|---------|----------------------|------------------------|--------|
| AW | IDLE → CALC → PASS → DONE | IDLE → CALC → SPLIT_FIRST → SPLIT_REMAIN → DONE | Wire-through |
| W | IDLE → COLLECT → OUTPUT | IDLE → LATCH → SERIALIZE | Wire-through |
| B | Wire-through | IDLE → CHECK → MERGE → OUTPUT | Wire-through |
| AR | 同 AW | 同 AW | Wire-through |
| R | IDLE → LATCH → LANE_STEER | IDLE → COLLECT → OUTPUT | Wire-through |

### 8.2 Bypass 條件

當 `AXI_DATA_WIDTH == 256` 時，所有 state machines 退化為 wire-through，Width Converter 不佔用任何 cycle。此為最常見的配置。

---

## 9. C++ Interface 設計

### 9.1 設計參數

```cpp
// 固定參數（來自 02_flit.md）
static constexpr int FLIT_WIDTH       = 408;
static constexpr int HEADER_WIDTH     = 56;
static constexpr int PAYLOAD_WIDTH    = 352;
static constexpr int NI_DATA_WIDTH    = 256;  // NI 側 AXI data width（固定）
static constexpr int AXI_ID_WIDTH     = 8;
static constexpr int AXI_ADDR_WIDTH   = 64;
static constexpr int ECC_WIDTH        = 32;

// Width Converter 配置參數
static constexpr int MAX_OUTSTANDING  = 8;    // 最大 outstanding transactions
```

### 9.2 Class Interface

```cpp
class WidthConverter {
public:
    // 建構時指定本地 AXI data width
    explicit WidthConverter(int local_axi_width);

    // Write path
    void process_aw(const AxiAwChannel& local_aw, std::vector<AxiAwChannel>& ni_aw_list);
    void process_w(const AxiWChannel& local_w, std::vector<AxiWChannel>& ni_w_list);
    void process_b(const AxiBChannel& ni_b, AxiBChannel& local_b, bool& local_b_valid);

    // Read path
    void process_ar(const AxiArChannel& local_ar, std::vector<AxiArChannel>& ni_ar_list);
    void process_r(const AxiRChannel& ni_r, AxiRChannel& local_r, bool& local_r_valid);

    // 查詢
    bool is_bypass() const { return local_width_ == NI_DATA_WIDTH; }
    int  get_ratio() const;
    bool is_upsizing() const { return local_width_ < NI_DATA_WIDTH; }
    bool is_downsizing() const { return local_width_ > NI_DATA_WIDTH; }

private:
    int local_width_;        // 本地 AXI data width
    int ratio_;              // 轉換比例

    // Upsizing pack buffer
    struct PackBuffer {
        uint8_t  data[32];   // 256-bit = 32 bytes
        uint8_t  strb[32];   // 32-byte wstrb
        int      beat_cnt;
    };

    // Downsizing serialize state
    struct SerializeState {
        uint8_t  data[128];  // 最大 1024-bit = 128 bytes
        uint8_t  strb[128];
        int      beat_cnt;
        int      total_beats;
    };

    // Split tracking
    struct SplitTracker {
        SplitInfo entries[MAX_OUTSTANDING];
        int       count;

        bool can_accept() const { return count < MAX_OUTSTANDING; }
        void allocate(uint8_t txn_id, const SplitInfo& info);
        void update_response(uint8_t txn_id, uint8_t resp);
        bool is_complete(uint8_t txn_id) const;
        SplitInfo release(uint8_t txn_id);
    };

    PackBuffer       w_pack_;
    SerializeState   w_ser_;
    SplitTracker     wr_split_tracker_;
    SplitTracker     rd_split_tracker_;

    // Internal helpers
    void calc_upsized_params(uint64_t addr, uint8_t len, uint8_t size,
                             uint64_t& new_addr, uint8_t& new_len, uint8_t& new_size);
    void calc_downsized_params(uint64_t addr, uint8_t len, uint8_t size,
                               std::vector<std::pair<uint64_t, uint8_t>>& sub_txns);
};
```

### 9.3 AXI Channel 結構

```cpp
struct AxiAwChannel {
    uint8_t  awid;
    uint64_t awaddr;
    uint8_t  awlen;
    uint8_t  awsize;
    uint8_t  awburst;
    uint8_t  awcache;
    uint8_t  awlock;
    uint8_t  awprot;
    uint8_t  awregion;
    uint8_t  awuser;
};

struct AxiWChannel {
    uint8_t  wdata[128];   // 最大 1024-bit (128 bytes)
    uint8_t  wstrb[128];
    bool     wlast;
    uint8_t  wuser;
};

struct AxiBChannel {
    uint8_t  bid;
    uint8_t  bresp;
    uint8_t  buser;
};

struct AxiArChannel {
    uint8_t  arid;
    uint64_t araddr;
    uint8_t  arlen;
    uint8_t  arsize;
    uint8_t  arburst;
    uint8_t  arcache;
    uint8_t  arlock;
    uint8_t  arprot;
    uint8_t  arregion;
    uint8_t  aruser;
};

struct AxiRChannel {
    uint8_t  rid;
    uint8_t  rdata[128];   // 最大 1024-bit (128 bytes)
    uint8_t  rresp;
    bool     rlast;
    uint8_t  ruser;
};
```

---

## 10. 實作考量

### 10.1 Buffer 需求

| 元件 | Upsizing | Downsizing | Bypass |
|------|----------|------------|--------|
| W Pack Buffer | ratio x local_width bits | 0 | 0 |
| W Serialize Buffer | 0 | local_width bits | 0 |
| R Lane Steer Buffer | 256 bits | 0 | 0 |
| R Reassembly Buffer | 0 | ratio x 256 bits | 0 |
| Split Tracker | 0 | MAX_OUTSTANDING entries | 0 |

### 10.2 Latency

| 轉換類型 | AW/AR Latency | W Latency | R Latency | B Latency |
|----------|---------------|-----------|-----------|-----------|
| Upsizing | 1 cycle（param calc） | ratio cycles → 1 NI beat | 1 NI beat → ratio cycles | 1 cycle（pass-through） |
| Bypass | 0 cycle | 0 cycle | 0 cycle | 0 cycle |
| Downsizing | 1+ cycle（param calc + split） | 1 local beat → ratio NI beats | ratio NI beats → 1 local beat | 1+ cycle（merge） |

### 10.3 Critical Paths

1. **AW/AR → Split 計算** — burst length 乘法與 256 比較
2. **W Serialize** — 選擇 data slice 的 MUX（local_width bits → 256 bits）
3. **R Reassembly** — 多個 256-bit beats 的累加器
4. **Response Merge** — precedence 比較邏輯

### 10.4 Address 對齊

Width Converter 處理以下對齊情況：

- **Upsizing**：本地 address 可能未對齊至 32-byte 邊界，需計算 start_offset 以確定在 256-bit data 中的起始 lane
- **Downsizing**：本地 address 已對齊至較寬 bus 寬度，拆分後各 sub-transaction 自然對齊至 32-byte 邊界

未對齊存取（narrow transfer within wide bus）透過 wstrb 處理，不需特殊邏輯。

### 10.5 Burst Type 限制

| Burst Type | Upsizing | Downsizing |
|------------|----------|------------|
| INCR（01） | 完整支援 | 完整支援（含 split） |
| WRAP（10） | 需轉換 wrap boundary | 需特殊處理（不建議 split） |
| FIXED（00） | 需重複同一 address | 不支援 split（每 beat 同 address） |

初始實作僅支援 INCR burst type。WRAP 和 FIXED burst type 的轉換屬於進階功能。

---

## 11. 設計模式

關鍵行為規則：
- **Burst length 計算**：`new_len = (original_len + 1) * ratio - align_adj - 1`
- **Response precedence**：`DECERR > SLVERR > EXOKAY > OKAY`
- **Lane steering**：基於 address alignment bits 選擇 data lane
- **Split 追蹤**：per-transaction state 用於 response merging

---

## 12. 待解決問題

- [ ] WRAP 和 FIXED burst type 的完整轉換支援
- [ ] 跨 split transaction 的 exclusive access（AxLOCK）語意保持
- [ ] 跨寬度邊界的 atomic operation 處理
- [ ] 跨 split transaction 的 QoS 維持策略
- [ ] Narrow transfer（awsize < bus width）的 byte lane 映射最佳化

---

## 相關文件

- [Flit Format](02_flit.md) — 固定參數定義與 payload 格式
- [Network Interface Specification](04_network_interface.md) — NI packetization 與 RoB
- [Router Specification](03_router.md) — 408-bit flit 轉發
