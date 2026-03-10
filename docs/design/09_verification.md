---
document_id: NOC-SPEC-09
title: Verification
version: 1.0
status: Draft
last_updated: 2026-03-09
prerequisite: [08_simulation.md]
---

# Verification

本文件定義 NoC CA Model 的驗證策略、Scoreboard 機制、效能指標收集與 RTL co-simulation 驗證方法。

---

## 1. Overview

```
                    ┌─────────────────────────────┐
                    │  RTL Co-simulation (DPI-C)  │  ← CA Model vs RTL 一致性
                    ├─────────────────────────────┤
                    │  System Integration Tests    │  ← 完整系統 end-to-end
                    ├─────────────────────────────┤
                    │  Scenario Integration Tests  │  ← 多元件交互場景
                    ├─────────────────────────────┤
                    │  Component Unit Tests        │  ← 單一元件隔離測試
                    └─────────────────────────────┘
```

---

## 2. 單元測試

使用 GoogleTest 框架。每個核心元件獨立測試。

### 2.1 元件測試矩陣

| Component | Test Focus | Min Cases |
|-----------|------------|:---------:|
| Flit Buffer | push/pop/full/empty/peek, depth boundary | 8 |
| Flit (Header) | pack/unpack round-trip, bit-field accuracy | 6 |
| XY Router | routing decision for all 5 output ports, edge cases | 12 |
| Path Lock | lock/unlock FSM, state transitions | 6 |
| QoS-Aware Arbiter | QoS priority, Round-Robin fairness | 10 |
| Crossbar | N×N switching, no-uturn, no-self-loop | 8 |
| Credit Counter | init/consume/release, underflow assert | 6 |
| NMU | flit packing (AW/W/AR), RoB allocate/release | 12 |
| NSU | flit unpacking, W reassembly, B/R response | 10 |
| ECC | generate/check round-trip, 1-bit correct, 2-bit detect | 6 |
| Address Translation | XY offset mode, SAM table mode | 4 |

---

## 3. 整合測試

### 3.1 關鍵場景

| ID | 場景 | 涉及元件 | 驗證重點 |
|----|------|---------|---------|
| IT-01 | 單次 32B Write (1 hop) | NMU→Router→NSU | 基本 end-to-end data integrity |
| IT-02 | 單次 64B Write (3 hop) | NMU→Router×3→NSU | Multi-hop routing correctness |
| IT-03 | 256B Burst Write (awlen=7) | NMU→Mesh→NSU | Wormhole lock/unlock, multi-flit |
| IT-04 | 單次 Read (AR→R) | NMU→Mesh→NSU→Mesh→NMU | 雙向 req/rsp 路徑 |
| IT-05 | Burst Read (arlen=15) | NMU→Mesh→NSU→Mesh→NMU | Multi-flit R response |
| IT-06 | 交叉流量衝突 | 2 NMU→同一 Router port | QoS arbitration, backpressure |
| IT-07 | Wormhole stall 傳播 | 長 burst + buffer 滿 | Backpressure propagation |
| IT-08 | Read-after-Write | Write 完成後 Read 同地址 | Data consistency |
| IT-10 | Max outstanding | 32 pending transactions | RoB full backpressure |
| IT-11 | Multi-ID reorder | 不同 axi_id 的 response OoO | RoB per-ID ordering |
| IT-12 | Credit exhaustion | 持續注入至 credit=0 | Flow control correctness |
| IT-13 | ECC single-bit error | 注入 1-bit error | ECC correction |
| IT-14 | ECC double-bit error | 注入 2-bit error | ECC detection, SLVERR |
| IT-14 | Width Converter | up/down conversion | Data integrity through converter |

---

## 4. Golden Verification

### 4.1 概念

Golden data 使用 **capture-at-source** approach：Transaction 發起時從 source memory 抓取 expected data，完成後從 destination memory 讀取 actual data 進行比對。

| Scenario | Golden Source | Capture Timing | Verification Target |
|----------|--------------|----------------|---------------------|
| Write | Source Memory | Transfer start | Destination Local Memory |
| Read | Source Local Memory | Transfer start | Response data (NMU RoB) |

### 4.2 Scoreboard

Online per-transaction tracking：submit 時記錄 expected outcome，response 到達時即時比對。

| Method | Description |
|--------|-------------|
| `register_expected(txn_handle, addr, data, len)` | Submit 時記錄 expected outcome |
| `check_actual(txn_handle, actual_data)` | Response 到達時即時比對 |
| `get_result(txn_handle)` | 取得單筆比對結果 |
| `get_summary()` | 整體 PASS/FAIL 統計 |

### 4.3 Write 驗證流程

```
Source Memory ──(1) Read + Capture──► Scoreboard (expected)
     │
     │ (2) Transfer via Mesh
     ▼
Destination Local Memory ──(3) Read actual──► Compare → PASS/FAIL
```

### 4.4 Read 驗證流程

```
Source Local Memory ──(1) Read + Capture──► Scoreboard (expected)
     │
     │ (2) Transfer via Mesh
     ▼
NMU RoB (response data) ──(3) Compare──► PASS/FAIL
```

### 4.5 Collision Detection

**規則：** 不同 source node 不得同時寫到同一 destination node 的同一 address。

偵測到 address collision 時產生 warning（不中斷 simulation）。同一 source node 的多次寫入是合法的（NMU injection order 保證順序）。

### 4.6 Verification Report

| Field | Description |
|-------|-------------|
| `passed` | 整體結果 |
| `total_checks` | 總驗證點數 |
| `passed_checks` | 通過數 |
| `failed_checks` | 失敗數 |
| `failures` | 失敗詳情 list |

### 4.7 驗證點

| 驗證點 | 位置 | 比較對象 |
|--------|------|---------|
| Write data integrity | NSU local memory | NMU 注入的原始資料 |
| Read data integrity | NMU 收到的 R response | NSU local memory 的實際資料 |
| Response ordering | NMU RoB release 順序 | Per-AXI-ID FIFO 順序 |
| ECC pass-through | Destination NI | Source NI 產生的 ECC 值 |

---

## 5. 效能指標

### 5.1 元件層級統計

| 元件 | Stats 類別 | 主要指標 |
|------|-----------|---------|
| Buffer | `BufferStats` | 讀寫次數、平均/最大佔用率 |
| Router | `RouterStats` | flit 數量、延遲、阻塞事件、per-port utilization |
| NI | `NIStats` | AXI 交易數、flit 數量 |
| Mesh | `MeshStats` | 總週期、總 flit 數 |
| Memory | `MemoryStats` | 讀寫次數、bytes 傳輸量 |

### 5.2 Metrics Collector

Metrics Collector 提供統一的效能指標收集介面，支援時間序列快照與延遲追蹤。

| Method | Description |
|--------|-------------|
| `capture()` | 每 cycle 擷取 snapshot |
| `record_injection(id, cycle)` | 記錄 flit 注入時間 |
| `record_ejection(id, cycle)` | 記錄 flit 彈出時間 |
| `get_throughput(start_cycle)` | 整體吞吐量 (bytes/cycle) |
| `get_latency_stats()` | 延遲統計 {min, max, avg, std} |
| `get_buffer_stats(capacity)` | Buffer 統計 {peak, avg, utilization} |

### 5.3 計算公式

| Metric | Formula |
|--------|---------|
| Throughput | `bytes_transferred / total_cycles`（以 AXI data 計量，32 bytes/flit） |
| Link Throughput (上限) | `FLIT_WIDTH / cycle = 400 bits/cycle = 50 bytes/cycle` |
| Latency | `ejection_cycle - injection_cycle` |
| Buffer Utilization | `avg_occupancy / total_capacity` |

### 5.4 效能基準測試

| 測試 | 量測指標 | 預期範圍 |
|------|---------|---------|
| Single flit latency (1 hop) | cycle count | 3-5 cycles |
| Single flit latency (max hop) | cycle count | MESH_COLS + MESH_ROWS + overhead |
| Burst throughput (no contention) | flits/cycle | ≈ 1.0 flit/cycle/port |
| Sustained throughput (uniform random) | flits/cycle/node | > 0.5 |
| Buffer utilization (high load) | avg / depth | > 0.7 |

---

## 6. RTL Co-simulation

### 6.1 架構

```
┌─────────────────┐       ┌─────────────────┐
│  SystemVerilog  │       │   CA Model       │
│  RTL Testbench  │◄─────►│  (via DPI-C)    │
│  RTL Router ×N  │       │  NoC System      │
│  RTL NI ×N     │       │                 │
└────────┬────────┘       └────────┬────────┘
         ▼                         ▼
    RTL Response              CA Model Response
         └────────► Comparator ◄───┘
                        │
                   PASS / FAIL
```

### 6.2 測試層級

**Match 基準：per-transaction。** 每筆 AXI transaction 的結果（data + response code）必須 CA Model 與 RTL 一致，不要求每 cycle 信號完全對齊。

| 層級 | 比較粒度 | 說明 |
|------|---------|------|
| Transaction-level | 比對完整 AXI transaction 結果（data + response code） | 主要驗證層級 |
| Latency-level | 比對 per-transaction latency | 僅 hardware pipeline mode，允許 ±5% tolerance |

### 6.3 Co-sim Test Plan

| # | 測試 | 說明 |
|---|------|------|
| 1 | Smoke test | Single write/read, CA Model vs RTL transaction result match |
| 2 | Stress test | 100 random transactions, all transaction results match |
| 3 | Corner cases | Buffer full, credit exhaustion |
| 4 | Performance | Latency/throughput within 5% tolerance |
| 5 | Reset behavior | Reset 後 state 歸零，新 traffic 正常運作 |
| 7 | Hot-swap drain | Channel drain 後替換元件，flit 不遺失 |

---

## 7. Coverage Targets

### 7.1 行為覆蓋率

| 類別 | 目標 | 量測方式 |
|------|------|---------|
| Code line coverage | ≥ 80% | gcov / llvm-cov |
| Branch coverage | ≥ 70% | gcov / llvm-cov |
| FSM state coverage | 100% | 所有 Path Lock 狀態轉移 |
| AXI channel coverage | 100% | 所有 5 channel 至少 1 transaction |
| Routing direction coverage | 100% | 所有 5 output port 至少 1 routing |
| Error path coverage | 100% | ECC error, credit boundary, buffer boundary |

### 7.2 功能覆蓋矩陣

| Feature | 相關測試 |
|---------|---------|
| XY Routing (all directions) | IT-01, IT-02 |
| Wormhole lock/unlock | IT-03, IT-07 |
| QoS arbitration | IT-06 |
| Credit flow control | IT-12 |
| RoB reorder | IT-09, IT-10 |
| ECC generate/check | IT-13, IT-14 |
| Backpressure propagation | IT-07, IT-12 |
| NMU injection buffer | IT-10 |
| NSU W-flit reassembly | IT-03 |
| Width Converter | IT-16 |

---

## 8. Related Documents

- [Router](03_router.md) — Router behavior under test
- [Network Interface](04_network_interface.md) — NI behavior under test
- [Simulation Platform](08_simulation.md) — NoC System API、DPI-C bridge

---

## Change Log

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-03-09 | Initial release |
