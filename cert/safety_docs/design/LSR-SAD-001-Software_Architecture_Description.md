# LSR-SAD-001: Software Architecture Description

| Document ID | LSR-SAD-001 |
|-------------|--------------|
| Version | 1.0 |
| Date | 2026-05-12 |
| Status | Draft |
| Classification | Safety-Critical |
| Standard | ISO 26262:2018 Part 6 |
| Target ASIL | ASIL D |

---

## Document Control

### Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | 2026-05-12 | Safety Team | Initial release |

### Referenced Documents

| Document ID | Title |
|-------------|-------|
| LSR-FSR-001 | Functional Safety Requirements |
| LSR-TSR-001 | Technical Safety Requirements |
| LSR-DS-001 | Design Specification |
| LSR-HSI-001 | Hardware-Software Interface |

---

## 1. Introduction

### 1.1 Purpose

This document describes the software architecture of the Luxoft Safe Renderer (LSR). It provides:
- System context and boundaries
- Layered architecture overview
- Component decomposition
- Interface definitions
- Data flow descriptions
- Safety architecture elements

### 1.2 Scope

This architecture description covers:
- Core engine components (`engine/`)
- Graphics interface layer (`gil/`)
- Platform interface layer (`pil/`)
- External interfaces (IHMI, DDH)

### 1.3 Architectural Goals

| Goal | Description | Priority |
|------|-------------|----------|
| Safety | Support ASIL D safety functions | Critical |
| Determinism | Bounded execution time, pre-allocated memory | Critical |
| Modularity | Clear component boundaries for testability | High |
| Portability | Hardware abstraction via GIL/PIL | High |
| Simplicity | Minimal complexity for safety certification | High |

---

## 2. System Context

### 2.1 Context Diagram

```
                    ┌─────────────────────────────────────┐
                    │         VEHICLE SYSTEM              │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │      HMI APPLICATION        │    │
                    │  │  (Customer Implementation)  │    │
                    │  └──────────────┬──────────────┘    │
                    │                 │ IHMI Interface    │
                    │                 ▼                   │
┌──────────────┐    │  ╔═════════════════════════════╗    │
│     DDH      │────┼──║   LUXOFT SAFE RENDERER      ║    │
│Configuration │    │  ║         (LSR)               ║    │
└──────────────┘    │  ╚═══════════════╤═════════════╝    │
                    │                  │                  │
                    │       ┌──────────┴──────────┐       │
                    │       ▼                     ▼       │
                    │  ┌─────────┐          ┌─────────┐   │
                    │  │   GIL   │          │   PIL   │   │
                    │  │(Graphics│          │(Platform│   │
                    │  │   HW)   │          │Services)│   │
                    │  └────┬────┘          └────┬────┘   │
                    │       │                    │        │
                    │       ▼                    ▼        │
                    │  ┌─────────┐          ┌─────────┐   │
                    │  │ Display │          │ System  │   │
                    │  │Hardware │          │ Timer   │   │
                    │  └─────────┘          └─────────┘   │
                    └─────────────────────────────────────┘
```

### 2.2 External Interfaces

| Interface | Direction | Description | ASIL |
|-----------|-----------|-------------|------|
| IHMI | Input | HMI application provides frame data | D |
| DDH | Input | Static configuration data | D |
| GIL | Output | Graphics rendering commands | D |
| PIL | Input/Output | Platform services (time, assertions) | D |
| Error | Output | Error status reporting | D |

### 2.3 System Boundary

**Inside System Boundary (Certified):**
- Engine core (`engine/lsr`)
- Database management (`engine/database`)
- Display management (`engine/display`)
- Frame handling (`engine/framehandler`)
- Common utilities (`engine/common`)

**Outside System Boundary (Integration Responsibility):**
- GIL implementation
- PIL implementation
- IHMI implementation
- DDH generation tool
- Hardware platform

---

## 3. Layered Architecture

### 3.1 Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    IHMI Interface                        │    │
│  │              (Customer HMI Application)                  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ENGINE LAYER                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Engine (Facade)                      │    │
│  │           render() | verify() | getError()              │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                    │
│  ┌──────────────┬───────────┼───────────┬──────────────────┐    │
│  │              │           │           │                   │    │
│  ▼              ▼           ▼           ▼                   │    │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐│    │
│ │ Database │ │ Display  │ │ Frame    │ │ Common Utilities ││    │
│ │ Module   │ │ Module   │ │ Handler  │ │ Module           ││    │
│ └──────────┘ └──────────┘ └──────────┘ └──────────────────┘│    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ABSTRACTION LAYER                              │
│  ┌───────────────────────┐    ┌───────────────────────────┐     │
│  │   Graphics Interface  │    │   Platform Interface      │     │
│  │     Layer (GIL)       │    │     Layer (PIL)           │     │
│  │  - Context management │    │  - Monotonic time         │     │
│  │  - Texture handling   │    │  - Assertion handling     │     │
│  │  - Rendering          │    │                           │     │
│  │  - Verification       │    │                           │     │
│  └───────────────────────┘    └───────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HARDWARE LAYER                                │
│  ┌───────────────────────┐    ┌───────────────────────────┐     │
│  │   Graphics Hardware   │    │   System Hardware         │     │
│  │  - GPU                │    │  - CPU                    │     │
│  │  - Frame buffer       │    │  - System timer           │     │
│  │  - Display            │    │  - Memory                 │     │
│  └───────────────────────┘    └───────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Layer Responsibilities

| Layer | Responsibility | Components |
|-------|----------------|------------|
| Application | Provide frame content via IHMI | Customer code |
| Engine | Orchestrate rendering and verification | Engine, Database, Display, FrameHandler |
| Abstraction | Hardware abstraction | GIL, PIL |
| Hardware | Physical rendering | GPU, Display, Timer |

### 3.3 Layer Coupling Rules

| Rule | Description |
|------|-------------|
| L1 | Upper layers may only call lower layers |
| L2 | Lower layers shall not call upper layers (no callbacks) |
| L3 | Components in same layer may communicate via defined interfaces |
| L4 | Cross-layer communication only through defined APIs |

---

## 4. Component Architecture

### 4.1 Component Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                          ENGINE MODULE                              │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                        Engine                               │    │
│  │  ┌─────────────────────────────────────────────────────┐   │    │
│  │  │ - m_db: Database                                     │   │    │
│  │  │ - m_display: DisplayManager                          │   │    │
│  │  │ - m_frameHandler: FrameHandler                       │   │    │
│  │  │ - m_error: LSREngineError                            │   │    │
│  │  └─────────────────────────────────────────────────────┘   │    │
│  │  + render(): bool                                          │    │
│  │  + verify(): bool                                          │    │
│  │  + handleWindowEvents(): bool                              │    │
│  │  + getError(): Error                                       │    │
│  └────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                        DATABASE MODULE                              │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                       Database                              │    │
│  │  + getBitmap(id): StaticBitmap*                            │    │
│  │  + getPanel(id): PanelType*                                │    │
│  │  + getFrame(id): FrameType*                                │    │
│  │  + getError(): LSREngineError                              │    │
│  └────────────────────────────────────────────────────────────┘    │
│  ┌────────────────────┐  ┌────────────────────┐                    │
│  │    StaticBitmap    │  │       Area         │                    │
│  │  + getData()       │  │  + x, y, w, h      │                    │
│  │  + getWidth()      │  └────────────────────┘                    │
│  │  + getHeight()     │  ┌────────────────────┐                    │
│  │  + getFormat()     │  │       Color        │                    │
│  └────────────────────┘  │  + r, g, b, a      │                    │
│                          └────────────────────┘                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                        DISPLAY MODULE                               │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                    DisplayManager                           │    │
│  │  + createWindow(): WindowCanvas*                           │    │
│  │  + getTexture(bitmap): Texture*                            │    │
│  │  + getError(): LSREngineError                              │    │
│  └────────────────────────────────────────────────────────────┘    │
│  ┌────────────────────┐  ┌────────────────────┐                    │
│  │      Texture       │  │    TextureCache    │                    │
│  │  + load()          │  │  + get(id)         │                    │
│  │  + isLoaded()      │  │  + size()          │                    │
│  └────────────────────┘  └────────────────────┘                    │
│  ┌────────────────────┐  ┌────────────────────┐                    │
│  │      Canvas        │  │   WindowCanvas     │                    │
│  │  + drawBitmap()    │  │  + swapBuffers()   │                    │
│  │  + clear()         │  └────────────────────┘                    │
│  │  + verify()        │                                            │
│  └────────────────────┘                                            │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                      FRAMEHANDLER MODULE                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                    FrameHandler                             │    │
│  │  + render(): bool                                          │    │
│  │  + verify(): bool                                          │    │
│  │  + getError(): LSREngineError                              │    │
│  └────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Widget Hierarchy                         │   │
│  │  ┌──────────┐                                               │   │
│  │  │  Window  │ (Root container)                              │   │
│  │  └────┬─────┘                                               │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  ┌──────────┐                                               │   │
│  │  │  Frame   │ (Mid-level container)                         │   │
│  │  └────┬─────┘                                               │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  ┌──────────┐                                               │   │
│  │  │  Panel   │ (Field container)                             │   │
│  │  └────┬─────┘                                               │   │
│  │       │                                                     │   │
│  │       ├───────────────┬──────────────────┐                  │   │
│  │       ▼               ▼                  ▼                  │   │
│  │  ┌──────────┐  ┌──────────────────┐  ┌───────────┐          │   │
│  │  │Bitmap    │  │ReferenceBitmap   │  │  Field    │          │   │
│  │  │Field     │  │Field             │  │  (Base)   │          │   │
│  │  │(Renders) │  │(Verifies)        │  │           │          │   │
│  │  └──────────┘  └──────────────────┘  └───────────┘          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                        COMMON MODULE                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Pool<T,N,A>    │  │ LSRErrorCollector│  │   Assertion      │  │
│  │  + allocate()    │  │  + setError()    │  │  + ASSERT()      │  │
│  │  + deallocate()  │  │  + getError()    │  │  + REQUIRE()     │  │
│  │  + isAllocated() │  └──────────────────┘  └──────────────────┘  │
│  │  + checkPool()   │  ┌──────────────────┐  ┌──────────────────┐  │
│  └──────────────────┘  │   LongTermPtr    │  │   ReturnValue    │  │
│  ┌──────────────────┐  │  + get()         │  │  + getValue()    │  │
│  │    PoolMarker    │  │  + isValid()     │  │  + isError()     │  │
│  │  + validate()    │  └──────────────────┘  └──────────────────┘  │
│  └──────────────────┘                                              │
└────────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Descriptions

#### 4.2.1 Engine Component

| Aspect | Description |
|--------|-------------|
| Purpose | Facade providing unified API to LSR functionality |
| Responsibilities | Orchestrate render/verify cycles; aggregate errors |
| Dependencies | Database, DisplayManager, FrameHandler |
| ASIL | D |

#### 4.2.2 Database Component

| Aspect | Description |
|--------|-------------|
| Purpose | Manage DDH configuration and bitmap resources |
| Responsibilities | Load/validate configuration; provide bitmap access |
| Dependencies | DDH data structures, Common utilities |
| ASIL | D |

#### 4.2.3 Display Component

| Aspect | Description |
|--------|-------------|
| Purpose | Manage graphics context and texture resources |
| Responsibilities | GIL context management; texture caching |
| Dependencies | GIL interface, Database |
| ASIL | D |

#### 4.2.4 FrameHandler Component

| Aspect | Description |
|--------|-------------|
| Purpose | Manage widget hierarchy and render traversal |
| Responsibilities | Widget tree management; render/verify coordination |
| Dependencies | Display, Database, Common utilities |
| ASIL | D |

#### 4.2.5 Common Component

| Aspect | Description |
|--------|-------------|
| Purpose | Provide safety-critical utilities |
| Responsibilities | Memory management; error handling; assertions |
| Dependencies | PIL (for pilAssert) |
| ASIL | D |

---

## 5. Data Flow

### 5.1 Render Data Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                        RENDER FLOW                                 │
│                                                                    │
│  ┌──────────┐                                                      │
│  │  IHMI    │ 1. getFrame()                                        │
│  └────┬─────┘                                                      │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────┐                                                      │
│  │  Engine  │ 2. render()                                          │
│  └────┬─────┘                                                      │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────┐                                                      │
│  │FrameHndlr│ 3. Traverse widget tree                              │
│  └────┬─────┘                                                      │
│       │                                                            │
│       ├─────────────────────────────────────────┐                  │
│       ▼                                         ▼                  │
│  ┌──────────┐                             ┌──────────┐             │
│  │ Database │ 4. getBitmap()              │ Display  │ 5. getTexture│
│  └────┬─────┘                             └────┬─────┘             │
│       │                                        │                   │
│       │ StaticBitmap                           │ Texture           │
│       └───────────────────┬────────────────────┘                   │
│                           ▼                                        │
│                    ┌──────────┐                                    │
│                    │  Canvas  │ 6. drawBitmap()                    │
│                    └────┬─────┘                                    │
│                         │                                          │
│                         ▼                                          │
│                    ┌──────────┐                                    │
│                    │   GIL    │ 7. gilDrawQuad()                   │
│                    └────┬─────┘                                    │
│                         │                                          │
│                         ▼                                          │
│                    ┌──────────┐                                    │
│                    │ Display  │ 8. gilSwapBuffers()                │
│                    │ Hardware │                                    │
│                    └──────────┘                                    │
└───────────────────────────────────────────────────────────────────┘
```

### 5.2 Verification Data Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                     VERIFICATION FLOW                              │
│                                                                    │
│  ┌──────────┐                                                      │
│  │  Engine  │ 1. verify()                                          │
│  └────┬─────┘                                                      │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────┐                                                      │
│  │FrameHndlr│ 2. Traverse widget tree                              │
│  └────┬─────┘                                                      │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────────────┐                                              │
│  │ReferenceBitmapFld│ 3. onVerify()                                │
│  └────┬─────────────┘                                              │
│       │                                                            │
│       ├───────────────────────────────────┐                        │
│       ▼                                   ▼                        │
│  ┌──────────┐                       ┌──────────┐                   │
│  │ Database │ 4. getRefBitmap()     │ Display  │ 5. getTexture()   │
│  └────┬─────┘                       └────┬─────┘                   │
│       │                                  │                         │
│       └─────────────┬────────────────────┘                         │
│                     ▼                                              │
│              ┌──────────┐                                          │
│              │  Canvas  │ 6. verify()                              │
│              └────┬─────┘                                          │
│                   │                                                │
│                   ▼                                                │
│              ┌──────────┐                                          │
│              │   GIL    │ 7. gilVerify()                           │
│              └────┬─────┘                                          │
│                   │                                                │
│                   ▼                                                │
│              ┌──────────┐                                          │
│              │ Compare  │ 8. Pixel comparison                      │
│              │ Pixels   │                                          │
│              └────┬─────┘                                          │
│                   │                                                │
│                   ▼                                                │
│              ┌──────────┐                                          │
│              │ Result   │ 9. true/false + error count              │
│              └──────────┘                                          │
└───────────────────────────────────────────────────────────────────┘
```

### 5.3 Error Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                        ERROR FLOW                                  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    Error Sources                          │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │     │
│  │  │Database│ │Display │ │ Frame  │ │ Pool   │ │  GIL   │  │     │
│  │  │ Error  │ │ Error  │ │Handler │ │ Error  │ │ Error  │  │     │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘  │     │
│  └──────┼──────────┼──────────┼──────────┼──────────┼────────┘     │
│         │          │          │          │          │              │
│         └──────────┴──────────┴──────────┴──────────┘              │
│                              │                                     │
│                              ▼                                     │
│                    ┌──────────────────┐                            │
│                    │ LSRErrorCollector │                           │
│                    │  (Aggregation)    │                           │
│                    └────────┬─────────┘                            │
│                             │                                      │
│                             ▼                                      │
│                    ┌──────────────────┐                            │
│                    │  Engine::m_error │                            │
│                    └────────┬─────────┘                            │
│                             │                                      │
│                             ▼                                      │
│                    ┌──────────────────┐                            │
│                    │ Engine::getError()│                           │
│                    └────────┬─────────┘                            │
│                             │                                      │
│                             ▼                                      │
│                    ┌──────────────────┐                            │
│                    │  Application     │                            │
│                    │  Error Handler   │                            │
│                    └──────────────────┘                            │
└───────────────────────────────────────────────────────────────────┘
```

---

## 6. Safety Architecture

### 6.1 Safety Mechanisms

```
┌───────────────────────────────────────────────────────────────────┐
│                    SAFETY MECHANISMS                               │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   MEMORY SAFETY                              │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │  Pool<T, Size>                                       │    │  │
│  │  │  - Pre-allocated memory (no runtime malloc)          │    │  │
│  │  │  - Marker-based corruption detection (0xAA/0x55)     │    │  │
│  │  │  - Bounds checking on all operations                 │    │  │
│  │  │  - Double-delete detection                           │    │  │
│  │  │  - Free list loop detection                          │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                 VIDEO OUTPUT VERIFICATION                    │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │  ReferenceBitmapField                                │    │  │
│  │  │  - Pixel-level comparison via gilVerify()            │    │  │
│  │  │  - Error counter for cumulative tracking             │    │  │
│  │  │  - Visibility-controlled activation                  │    │  │
│  │  │  - 99%+ diagnostic coverage                          │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   ERROR DETECTION                            │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │  LSRErrorCollector                                   │    │  │
│  │  │  - Hierarchical error aggregation                    │    │  │
│  │  │  - Severity-based retention                          │    │  │
│  │  │  - Domain-specific error codes                       │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │  Assertion Framework                                 │    │  │
│  │  │  - ASSERT for debug-time checks                      │    │  │
│  │  │  - REQUIRE for runtime validation                    │    │  │
│  │  │  - pilAssert callback for platform handling          │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                  DATA VALIDATION                             │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │  Configuration Validation                            │    │  │
│  │  │  - DDH magic number verification                     │    │  │
│  │  │  - DDH version checking                              │    │  │
│  │  │  - Bitmap ID range validation                        │    │  │
│  │  │  - Pointer NULL checks                               │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

### 6.2 ASIL Decomposition

| Component | ASIL | Rationale |
|-----------|------|-----------|
| Engine | D | Top-level orchestrator; all safety goals |
| Database | D | Data integrity affects rendering correctness |
| DisplayManager | D | Texture management affects rendering |
| FrameHandler | D | Widget rendering and verification |
| Pool | D | Memory safety foundational to all operations |
| ReferenceBitmapField | C | Verification mechanism (SG4) |
| Canvas | D | Rendering commands |
| GIL Interface | D | Graphics output (integration responsibility) |
| PIL Interface | D | Platform services (integration responsibility) |

### 6.3 Freedom from Interference

| Mechanism | Description |
|-----------|-------------|
| Memory Isolation | Each Pool instance is separate; no shared storage |
| Error Isolation | Component errors don't propagate to corrupt other components |
| Interface Contracts | Clear APIs prevent unintended interactions |
| Const Correctness | Read-only DDH prevents modification |

---

## 7. Interface Specifications

### 7.1 IHMI Interface

```cpp
class IHMI
{
public:
    virtual Frame* getFrame() = 0;
};
```

| Method | Description | ASIL |
|--------|-------------|------|
| getFrame() | Returns current frame to render | D |

### 7.2 Engine Public Interface

```cpp
class Engine
{
public:
    Engine(const DDHType* ddh, IHMI& hmi);
    bool render();
    bool verify();
    bool handleWindowEvents();
    Error getError();
};
```

### 7.3 GIL Interface Summary

See LSR-HSI-001 for complete GIL interface specification.

| Function | Purpose | ASIL |
|----------|---------|------|
| gilCreateContext() | Create rendering context | D |
| gilCreateWindow() | Create window surface | D |
| gilSetSurface() | Bind rendering target | D |
| gilCreateTexture() | Create texture object | D |
| gilTexPixels() | Load texture data | D |
| gilDrawQuad() | Render textured quad | D |
| gilVerify() | Compare pixels against reference | C |
| gilSwapBuffers() | Present frame | D |
| gilGetError() | Retrieve error status | D |

### 7.4 PIL Interface Summary

```cpp
extern "C" {
    uint32_t pilGetMonotonicTime(void);
    void pilAssert(const char* msg, const char* file, int32_t lineNo);
}
```

| Function | Purpose | ASIL |
|----------|---------|------|
| pilGetMonotonicTime() | Get system time in milliseconds | C |
| pilAssert() | Handle assertion failures | D |

---

## 8. Deployment View

### 8.1 Static Library Structure

```
liblsr.a
├── engine/lsr/
│   └── Engine.o
├── engine/database/
│   ├── Database.o
│   ├── Area.o
│   └── LsrImage.o
├── engine/display/
│   ├── DisplayManager.o
│   ├── Canvas.o
│   ├── WindowCanvas.o
│   ├── Texture.o
│   └── TextureCache.o
├── engine/framehandler/
│   ├── FrameHandler.o
│   ├── Widget.o
│   ├── Window.o
│   ├── Frame.o
│   ├── Panel.o
│   ├── Field.o
│   ├── BitmapField.o
│   └── ReferenceBitmapField.o
└── engine/common/
    └── Assertion.o

libgil.a (implementation-specific)
└── gil.o

libpil.a (platform-specific)
└── pil.o
```

### 8.2 Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                      MEMORY MAP                                  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  CODE SECTION (.text)                                      │  │
│  │  - Engine functions                                        │  │
│  │  - Database functions                                      │  │
│  │  - Display functions                                       │  │
│  │  - FrameHandler functions                                  │  │
│  │  - Common utilities                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  READ-ONLY DATA SECTION (.rodata)                          │  │
│  │  - DDH configuration (const)                               │  │
│  │  - Bitmap pixel data (const)                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  DATA SECTION (.data/.bss)                                 │  │
│  │  - Engine instance                                         │  │
│  │  │  - Database member                                      │  │
│  │  │  - DisplayManager member                                │  │
│  │  │  - FrameHandler member                                  │  │
│  │  │  - Error state                                          │  │
│  │  - Pool storage (pre-allocated)                            │  │
│  │  │  - Widget pool                                          │  │
│  │  │  - Texture pool                                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  STACK                                                     │  │
│  │  - Function call frames                                    │  │
│  │  - Local variables                                         │  │
│  │  - (Bounded recursion)                                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  NO HEAP ALLOCATION                                        │  │
│  │  (malloc/new not used at runtime)                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Design Decisions

### 9.1 Key Architectural Decisions

| ID | Decision | Rationale | Alternatives Considered |
|----|----------|-----------|-------------------------|
| AD-01 | Facade pattern for Engine | Single entry point simplifies API and error management | Multiple entry points |
| AD-02 | Composite pattern for widgets | Natural tree structure matches HMI hierarchy | Flat widget list |
| AD-03 | Template-based pools | Type safety with compile-time size validation | Runtime-sized pools |
| AD-04 | C interface for GIL/PIL | Maximum portability; SEooC boundary | C++ interface |
| AD-05 | Marker-based corruption detection | Simple, deterministic detection mechanism | CRC-based detection |
| AD-06 | No heap allocation | Deterministic memory behavior | Dynamic allocation with monitoring |

### 9.2 Design Constraints

| Constraint | Impact | Source |
|------------|--------|--------|
| No dynamic allocation | Pre-sized pools; fixed widget counts | ASIL D determinism |
| Bounded execution | O(n) algorithms only; no unbounded loops | ASIL D timing |
| C interface for portability | GIL/PIL are C interfaces | SEooC boundary |
| Const DDH data | Configuration immutable at runtime | Data integrity |

---

## 10. Traceability

### 10.1 Architecture to Requirements

| Component | Related FSRs |
|-----------|--------------|
| Engine | FSR-AV-001, FSR-AV-002, FSR-AV-003, FSR-ER-001, FSR-IN-001 |
| Database | FSR-DD-001, FSR-DD-002, FSR-DD-003, FSR-DD-004 |
| DisplayManager | FSR-DD-005, FSR-TI-003 |
| FrameHandler | FSR-DD-005, FSR-AV-001, FSR-AV-004 |
| ReferenceBitmapField | FSR-VE-001, FSR-VE-002, FSR-VE-003, FSR-VE-004 |
| Pool | FSR-MS-001, FSR-MS-002, FSR-MS-003, FSR-MS-004, FSR-MS-005 |
| LSRErrorCollector | FSR-ER-001, FSR-ER-002 |
| Assertion | FSR-ER-003 |

---

**End of Document**
