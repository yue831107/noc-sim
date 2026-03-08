# Design Specification 撰寫規範

## 核心原則

1. **Spec 描述 WHAT，不描述 HOW** — 定義行為與規格，不寫實作細節
2. **一件事只寫一次** — 避免跨文件重複。若需引用，用連結指向 single source of truth
3. **結構一致** — 同類文件遵循相同章節框架，讀者能預期在哪找到什麼
4. **最小化 code** — Spec 中不放大段 C++ / Python code。允許：signal table、短 pseudocode（≤10 行）、struct 欄位定義。禁止：完整 class 實作、function body、algorithm implementation
5. **圖優於文字** — 用 ASCII block diagram、timing diagram、state diagram 取代長篇文字描述

## 章節框架

以下為**元件規格文件**（Router、NI 等）的標準章節。非元件文件（flit format、simulation、verification）可調整，但應保持有條理的層次。

### 1. Overview
- 元件用途（1~2 段）
- 在系統中的位置（簡述上下游關係）
- 設計目標與限制

### 2. Architecture & Block Diagram
- ASCII block diagram（必須）
- 主要子模組列表與職責（表格）
- 資料流方向標示

### 3. Parameters
- 可配置參數表（名稱、型別、預設值、說明、來源）
- 參數之間的約束關係
- 指向 NocConfig 的 single source of truth

### 4. Interface Specification
- 外部 port 信號表（名稱、寬度、方向、說明）
- Handshake 協議（timing diagram）
- 與相鄰元件的連接關係

### 5. Functional Requirements
- 逐條列出功能需求（FR-xx 編號）
- 每條需求：描述、前置條件、預期行為、例外處理
- 用 state diagram 或 timing diagram 輔助說明
- **不寫 implementation algorithm** — 只描述 input → output 的行為規格

### 6. Performance
- Latency 規格（best case / worst case）
- Throughput 規格
- Resource 使用（buffer 深度、port 數量等）

### 7. 相關文件
- 連結到相關的 spec 文件

## 格式規則

### 表格優先
- 參數 → 表格
- 信號 → 表格
- 狀態轉移 → 表格或 state diagram
- 比較 → 表格

### Diagram 規則
- 使用 ASCII art（不依賴外部工具）
- Block diagram 標示：模組名、port 名、資料流箭頭方向
- Timing diagram 標示：cycle 編號、signal 名

### 禁止事項
- 不在 spec 中放完整 class / function 實作（放在 code 裡）
- 不重複定義已在其他文件定義的內容（用連結）
- 不放外部 IP 名稱（FlooNoC、booksim2、pcievhost 等）
- 不在多個文件重複同一張參數表（指向 single source）
- 不用模糊用詞（「可能」「大概」「通常」）— 規格必須明確

### 允許的 Code
- Signal / struct **欄位定義**（≤ 10 行，展示資料結構，不含 method body）
- 短 **pseudocode**（≤ 10 行，描述行為邏輯，不是可編譯的 code）
- **Enum 定義**（列舉狀態或模式）

### 用詞約定
- 「SHALL」= 強制要求
- 「SHOULD」= 建議
- 「MAY」= 可選
- 參數名用 `backtick`
- 首次出現的縮寫需展開全稱

## 非元件文件的調整

| 文件類型 | 調整方式 |
|---------|---------|
| Flit Format | 以 bit field 表格為主，不需 Architecture 章節 |
| Simulation | 以 API 分組 + I/O pattern 為主，可有較多 pseudocode |
| Verification | 以 test plan 表格為主，不需 Performance 章節 |
| Memory Ops | 以 operation flow 為主，用 sequence diagram |
| QoS / Multicast | 以 policy 規則為主，用 decision table |
