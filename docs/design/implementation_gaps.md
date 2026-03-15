---
document_id: NOC-REF-GAPS
title: Implementation Gaps & Risk Register
version: 1.0
status: Active
last_updated: 2026-03-10
prerequisite: []
---

# Implementation Gaps & Risk Register

本文件列出設計文件審查後發現的實作缺口、風險點與待釐清的設計決策，供後續實作階段參考。

---

## High Priority

### H-1: AXI Slave Stall 不模擬

- **所在文件**: `08_simulation.md` §11.3
- **說明**: 明確標記「AXI slave 總是就緒」，但真實系統中 memory controller 有延遲。若 RTL 有 stall，co-sim 必定失敗。
- **影響**: Co-simulation 正確性
- **待決策**: 是否實作可配置 stall model（fixed latency / random stall / stall pattern）

### H-2: NMU Injection Ordering AW→W 保證

- **所在文件**: `03_router.md` FR-02, `04_network_interface.md` FR-04
- **說明**: 文件說「AW→W ordering 由 NMU 注入順序保證」，但 NMU 具體如何保證（同 injection buffer? 嚴格 FIFO?）未見明確 state machine 或 constraint 描述。
- **影響**: 核心 correctness — AW 必須先於對應 W 到達 NSU
- **待決策**: 定義 NMU injection state machine（AW→W sequential injection constraint）

### H-3: Deadlock Detection Forward Progress 定義

- **所在文件**: `08_simulation.md` §5.4
- **說明**: `DEADLOCK_THRESHOLD` 以「至少一個 Router input buffer pop」判斷 forward progress。但若 NI injection buffer 持續接受 flit 而 Router 未 pop（例如 NI→Router Channel 延遲中），可能誤判為 deadlock。
- **影響**: Simulation 穩定性
- **待決策**: 將 forward progress 定義擴展至包含 NI buffer activity，或增加 grace period

### H-4: Hot-Swap Drain 機制

- **所在文件**: `08_simulation.md` §9.6
- **說明**: 約束提到「必須 drain Channel pipeline」，但未定義 drain 的 API（`drain_channel()`?）、判定條件（`channel.empty()`?）、或自動化 drain 流程。
- **影響**: Hot-Swap 功能可用性
- **待決策**: 定義 drain API 與判定條件

---

## Medium Priority

### M-1: ~~QoS Regulator response_bytes 來源~~ [RESOLVED]

- **所在文件**: `06_qos.md` §2.4, §7.3
- **說明**: 原始設計中 Regulator 需追蹤 response bytes，依賴 `on_response(bytes)` callback。重寫後 Regulator 改為追蹤 request bytes（NMU 注入時即可計算），不再需要 response callback。
- **影響**: 已解決 — callback 來源問題不再存在
- **狀態**: **RESOLVED**（參見 06_qos.md §2.4 與 §7.3）

### M-2: Multi-port LOCAL no-UTurn 規則 Use Case

- **所在文件**: `03_router.md` FR-08
- **說明**: 說「L0→L1 inter-port 直連允許」，但 routing function 中 `dst == cur` 時轉 LOCAL、`port_id` 選具體 port，何時會出現 L0→L1 的需求？需釐清 use case。
- **影響**: 設計清晰度
- **待決策**: 確認 L0↔L1 inter-port routing 的實際場景（Gateway Host DMA ↔ Local PE?）

### M-3: Width Converter 與 NI Phase 整合

- **所在文件**: `10_width_converter.md` §2
- **說明**: Implementation Note 提到「在 NI tick() 之前執行」，但在 8-phase model 中具體屬於哪個 phase 未定義。
- **影響**: Simulation 時序正確性
- **待決策**: 歸入 Phase 8 開頭或定義為 Phase 0（pre-NI）

### M-4: SAM Table 格式與載入

- **所在文件**: `04_network_interface.md` FR-01
- **說明**: `sam_rule_t` 提到但未定義具體欄位（base_addr? mask? dst_id?）。`USE_ID_TABLE` 模式缺少具體配置範例。
- **影響**: Address Translation 實作
- **待決策**: 定義 `sam_rule_t` 結構與 JSON 配置格式

### M-5: Traffic Pattern inject_cycle "after:" 語法

- **所在文件**: `08_simulation.md` §4.3
- **說明**: `inject_cycle` 支援 `"after:<txn_id>"` 依賴語法，但 Traffic Manager 如何解析並排程未描述。
- **影響**: Traffic Manager 實作
- **待決策**: 定義依賴解析演算法（topological sort? runtime check?）

### M-6: ECC Hsiao Matrix 定義

- **所在文件**: `02_flit.md` §3.6, `04_network_interface.md` FR-06
- **說明**: 提到「Hsiao SECDED」但未定義具體 H-matrix。若 CA Model 與 RTL 使用不同 H-matrix，ECC 值將不匹配。
- **影響**: Co-simulation ECC 一致性
- **待決策**: 提供具體 H-matrix 或引用標準定義

### M-7: Scoreboard Collision Detection 粒度

- **所在文件**: `09_verification.md` §4.5
- **說明**: 偵測「不同 source node 寫同一 destination 同一 address」，但 address 比較粒度未定義（byte? burst range?）。Burst write 的 address range 計算規則？
- **影響**: Scoreboard 正確性
- **待決策**: 定義 collision detection 的 address range 計算（base_addr + burst_size × burst_len?）

### M-8: NI 的 `post_wire()` 缺失

- **所在文件**: `08_simulation.md` §6, `00_architecture.md` §7.6
- **說明**: Router 有 `tick()` + `post_wire()`，但 NI 只有 `tick()`。NI 在 Phase 6-7 是否需要 clear accepted / credit update？
- **影響**: NI flow control 正確性
- **待決策**: 確認 NI 是否需要 `post_wire()` 或其 Phase 6-7 邏輯已包含在 Phase 8

---

## Low Priority

### L-1: Age-Based Promotion 實作範圍

- **所在文件**: `03_router.md` FR-04, `06_qos.md` §6.1
- **說明**: 多處提到「optional」age-based promotion，但未定義 age counter 寬度、promotion 閾值、promotion step。
- **待決策**: 定義 age-based promotion 參數或標記為 V2 功能

### L-2: Per-QoS VC 對映

- **所在文件**: `03_router.md` FR-04, `06_qos.md` §6.1
- **說明**: 提到「未來擴展」，但 `NUM_VC` 與 QoS level 的對映策略完全未定義。
- **待決策**: 定義 VC-QoS mapping 策略或標記為 V2

### L-3: Cycle Trace 格式

- **所在文件**: `08_simulation.md` §4.6
- **說明**: 僅提到「VCD 或 JSON trace 格式」，未定義具體 signal 名稱、hierarchy、sampling 條件。
- **待決策**: 定義 trace signal list 與 hierarchy

### L-4: Statistics JSON Schema

- **所在文件**: `08_simulation.md` §4.7
- **說明**: 未定義 output JSON 的具體 schema（key names, nesting structure）。
- **待決策**: 提供 JSON schema 或範例 output

### L-5: Reset 行為矛盾

- **所在文件**: `08_simulation.md` §11.1, `09_verification.md` §6.3 #6
- **說明**: `08_simulation.md` 標記「不模擬 reset sequence」，但 co-sim test plan（`09_verification.md` §6.3 #6）包含「Reset behavior」測試。二者矛盾。
- **待決策**: 統一定義 — 移除 co-sim reset test 或加入最小 reset 模擬

### L-6: BufferPolicy Pluggable 未定義

- **所在文件**: `PROJECT_GOALS.md` §3.4
- **說明**: 提到 `BufferPolicy` abstract base（Private/Shared），但所有 Router 相關文件未提及 shared buffer 概念。
- **待決策**: 移除 `BufferPolicy` 或補充 shared buffer 設計

### L-7: 02_flit.md 缺少 §1.2

- **所在文件**: `02_flit.md`
- **說明**: Section 編號從 1.1 跳到 1.3，缺少 §1.2。
- **狀態**: 已修正（見 Change Log）

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-10 | Initial release — 5 High, 8 Medium, 7 Low gaps identified |
