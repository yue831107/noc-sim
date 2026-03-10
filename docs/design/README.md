# NoC Cycle-Approximate Model - Design Documents

本目錄包含 NoC Cycle-Approximate Model 的設計規格文件。

---

## 文件層級

| Level | Document | Target Audience |
|-------|----------|-----------------|
| 0 | [Project Goals](../PROJECT_GOALS.md) | PM / System Architect |
| 1 | [System Architecture](00_architecture.md) | Hardware Architect |
| 2 | 01~10 Component Specs | RTL / C++ Engineer |
| 3 | [User Guide](noc_model_guide.md) | End User / Verification Engineer |

---

## 文件索引

### Architecture Overview

| # | Document | Description |
|---|----------|-------------|
| 00 | [System Architecture](00_architecture.md) | **Entry point** — architecture diagrams, data flow, HW/SW interface, I/O patterns, co-sim |

### Core — 01~04

| # | Document | Description |
|---|----------|-------------|
| 01 | [System Overview](01_overview.md) | Topology parameters |
| 02 | [Flit Format](02_flit.md) | **Baseline** — fixed 400-bit flit, Header/Payload format |
| 03 | [Router](03_router.md) | Ports, XY Routing, Wormhole, Pipeline |
| 04 | [Network Interface](04_network_interface.md) | NMU/NSU, AXI ↔ Flit conversion, RoB, ECC |

### Architecture — 05~08

| # | Document | Description |
|---|----------|-------------|
| 05 | [Physical Channel](05_physical_channel.md) | Dual Req/Rsp channel architecture, HoL Blocking analysis |
| 06 | [QoS Design](06_qos.md) | QoS priority design, Generator, Probe |

### Simulation & Verification — 08~09

| # | Document | Description |
|---|----------|-------------|
| 08 | [Simulation Platform](08_simulation.md) | NocConfig, API (5 Groups), Cycle Model, Channel\<T\>, Hot-Swap, DPI-C |
| 09 | [Verification](09_verification.md) | Scoreboard, performance metrics, test strategy, coverage |

### Appendix

| # | Document | Description |
|---|----------|-------------|
| 10 | [Width Converter](10_width_converter.md) | AXI width conversion (32b~1024b ↔ 256b) |
| — | [Implementation Gaps](implementation_gaps.md) | Implementation gaps & risk register (High/Medium/Low) |

---

## 圖片資源

| File | Content |
|------|---------|
| [platform_diagram.svg](../images/platform_diagram.svg) | Platform overview (User Interface / NoC System Internals / Golden Verification) |
| [hot_swap.svg](../images/hot_swap.svg) | Hot-Swap concept (CA ↔ RTL switching) |

---

## 中英文混用規範

本專案文件遵循以下中英文混用規範：

| Context | Language | Example |
|---------|----------|---------|
| Paragraph text | 繁體中文 | "每個 Router 由 Req Router 與 Rsp Router 兩個結構對稱的子元件組成" |
| Table column headers | English | `Parameter`, `Default`, `Description` |
| Table Description column | English | Consistent with code and technical terms |
| Function / API names | English code font | `tick()`, `process_cycle()` |
| ASCII diagram labels | English | Block names, signal names inside diagrams |
| Commit messages | English | `docs: fix API group count` |

---

## Parameter Single Source of Truth

| Parameter Category | Authoritative Document |
|--------------------|----------------------|
| Flit format | [02_flit.md](02_flit.md) §1.1 |
| NocConfig configurable parameters | [08_simulation.md](08_simulation.md) §3.1 |
| QoS CSR memory map | [06_qos.md](06_qos.md) §4.1 |
| Router pipeline stages | [03_router.md](03_router.md) §3 |
| NI functional requirements | [04_network_interface.md](04_network_interface.md) §2 |
