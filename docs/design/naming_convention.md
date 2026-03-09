---
document_id: NOC-REF-NC
title: Naming Convention Reference
version: 1.0
status: Active
last_updated: 2026-03-09
prerequisite: []
---

# Naming Convention Reference

本文件定義 NoC 專案的統一命名規範，對齊業界商業 NoC IP 慣例（Arteris FlexNoC、Arm CMN、AMD Versal、Cadence Janus）。

---

## 1. Model Abstraction Naming

| Canonical Name | Abbreviation | Description | Industry Alignment |
|----------------|-------------|-------------|-------------------|
| **Cycle-Approximate Model** | **CA Model** | 本專案的 C++ 模擬模型，8-phase cycle-approximate | Arteris Architectural View |
| **RTL** | — | Register Transfer Level 硬體描述 | 全行業統一 |

**禁止使用**：~~virtual model~~、~~C++ behavior model~~、~~C++ model~~

**文件標題**：`NoC Cycle-Approximate Model`（取代原 `NoC Virtual Behavior Model`）

---

## 2. Simulation Mode Naming

| Canonical Name | Description | Industry Alignment |
|----------------|-------------|-------------------|
| **Standalone Simulation** | 全部元件為 CA 實作，不接 RTL | Arm Fast Model standalone |
| **NI RTL Co-Simulation** | Router 為 CA，NI 替換為 RTL | — |
| **Router RTL Co-Simulation** | NI 為 CA，Router 替換為 RTL | — |
| **Full RTL Co-Simulation** | 全部 Router + NI 替換為 RTL | — |

**禁止使用**：~~Pure Virtual~~、~~Pure Virtual Simulation~~

---

## 3. Component Naming

### 3.1 Core Components

| Canonical Name | Abbreviation | Code Class Name | Description |
|----------------|-------------|-----------------|-------------|
| Router | — | `CppRouter` | Mesh 封包轉發單元 |
| Network Interface | NI | `CppNI` | AXI ↔ Flit 轉換單元 |
| NoC Master Unit | NMU | — | NI 中的 AXI Slave 端（接收 Master 請求） |
| NoC Slave Unit | NSU | — | NI 中的 AXI Master 端（驅動 Slave 介面） |
| Req Router | — | `ReqRouter` | Request 子 Router（AW/W/AR） |
| Rsp Router | — | `RspRouter` | Response 子 Router（B/R） |
| Crossbar | — | `Crossbar` | N×N switch fabric |
| Reorder Buffer | RoB | — | Per-NMU response reordering |

**禁止使用**：~~Virtual Router~~、~~Virtual NI~~、~~sub-router~~、~~子 Router~~

**Note**：在 Hot-Swap 語境中需區分 CA 與 RTL 實作時，使用「CA Router」/「RTL Router」及「CA NI」/「RTL NI」。

### 3.2 Hot-Swap Components

| CA Implementation | DPI-C Bridge | Abstract Interface |
|-------------------|-------------|-------------------|
| CA Router (`CppRouter`) | Router DPI Bridge (`RouterDpiBridge`) | `Router_Interface<Mode>` |
| CA NI (`CppNI`) | NI DPI Bridge (`NIDpiBridge`) | `NI_Interface<Mode>` |

### 3.3 Other Components

| Canonical Name | Code Class Name |
|----------------|-----------------|
| Traffic Manager | `TrafficManager` |
| Scoreboard | `Scoreboard` |
| Metrics Collector | `MetricsCollector` |
| Channel\<T\> | `Channel<T>` |
| Input Buffer | `InputBuffer` |
| Buffer State | `BufferState` |
| Route Computer | `RouteCompute` |
| Path Lock | `PathLock` |
| Allocator | `Allocator` |
| Host Memory | `HostMemory` |
| Local Memory | `LocalMemory` |

---

## 4. Capitalization Rules

### 4.1 Component & Abbreviation

| Category | Rule | Examples |
|----------|------|---------|
| Component abbreviation | ALL-CAPS | NMU, NSU, NI, VC, QoS, ECC, RoB, NoC |
| Component full name | Title Case | Network Interface, Reorder Buffer, Packet Switch |
| Simulation mode | Title Case | Standalone Simulation, RTL Co-Simulation |
| NoC | Fixed form | NoC（不是 NOC 或 noc） |
| Namespace | lowercase | `noc::` |

### 4.2 Signal Naming

| Context | Rule | Examples |
|---------|------|---------|
| Protocol signal（文件中） | ALLCAPS（依 AMBA spec） | VALID, READY, AWVALID, AWREADY |
| Flow control type（文件中） | Title Case + slash | Valid/Ready, Credit-Based |
| Code signal name | snake_case | `valid_i`, `ready_o`, `credit_cnt` |

### 4.3 Documentation

| Context | Rule | Examples |
|---------|------|---------|
| Heading | Title Case | "Router Architecture", "Flow Control Modes" |
| Body text | lowercase（除專有名詞外） | "the router forwards...", "co-simulation with RTL" |
| AXI channel name | ALLCAPS | AW, W, AR, B, R |
| AXI signal name | lowercase（依 AXI spec） | `awlen`, `arsize`, `wstrb` |

---

## 5. Abbreviation First-Occurrence Rule

每份文件第一次出現縮寫時，使用完整格式：

```
Network Interface (NI)
NoC Master Unit (NMU)
NoC Slave Unit (NSU)
Reorder Buffer (RoB)
Virtual Channel (VC)
Quality of Service (QoS)
Error Correction Code (ECC)
```

後續使用縮寫即可。

---

## 6. Cross-Reference: Industry Vendor Terminology

| Our Term | Arteris | Cadence | Arm | AMD/Xilinx |
|----------|---------|---------|-----|-----------|
| CA Model | Architectural View | Functional SystemC Model | Performance Model | SystemC TLM Model |
| RTL | RTL | RTL | RTL | RTL |
| Standalone Simulation | — | — | Fast Model standalone | TLM Simulation |
| RTL Co-Simulation | Verification View | Mixed Simulation | Cycle Model | System Simulation |
| Router | Switch | Routing Node | XP (Crosspoint) | NPS |
| NI | NIU | IEA / TEA | ASIB / AMIB | — |
| NMU | Initiator NIU | IEA | RN (Request Node) | NMU |
| NSU | Target NIU | TEA | SN (Subordinate Node) | NSU |

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release — industry-aligned naming convention |
