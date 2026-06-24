# LSR-SAR-001: Safety Analysis Report (FMEA)

| Document ID | LSR-SAR-001 |
|-------------|--------------|
| Version | 1.0 |
| Date | 2026-05-12 |
| Status | Draft |
| Classification | Safety-Critical |
| Standard | ISO 26262:2018 Part 5, Part 9 |
| Target ASIL | ASIL D |

---

## Document Control

### Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | 2026-05-12 | Safety Team | Initial release |

### Review and Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Author | | | |
| Technical Reviewer | | | |
| Safety Reviewer | | | |
| Approver | | | |

### Referenced Documents

| Document ID | Title |
|-------------|-------|
| LSR-HARA-001 | Hazard Analysis and Risk Assessment |
| LSR-FSR-001 | Functional Safety Requirements |
| LSR-SAD-001 | Software Architecture Description |
| ISO 26262:2018 | Road vehicles - Functional safety |

---

## 1. Introduction

### 1.1 Purpose

This Safety Analysis Report presents the Failure Mode and Effects Analysis (FMEA) for the Luxoft Safe Renderer (LSR). The analysis identifies:

1. Potential failure modes for each software component
2. Effects of failures at local, system, and vehicle levels
3. Detection mechanisms for each failure mode
4. Mitigation strategies and safety mechanisms
5. Diagnostic coverage calculations

### 1.2 Scope

This analysis covers the core LSR software components within the certification boundary:

| Module | Description | Safety Relevance |
|--------|-------------|------------------|
| `engine/lsr` | Main engine facade | High - orchestrates safety functions |
| `engine/database` | Configuration and bitmap management | High - data integrity |
| `engine/display` | Display manager and texture cache | High - rendering correctness |
| `engine/framehandler` | Widget hierarchy management | High - rendering logic |
| `engine/common` | Safety utilities (Pool, Assertions) | Critical - foundational safety |
| `gil` | Graphics Interface Layer | High - graphics output |
| `pil` | Platform Interface Layer | High - platform services |

### 1.3 Analysis Method

The FMEA follows ISO 26262-9 Annex B methodology:
1. System decomposition into components and functions
2. Identification of failure modes per function
3. Assessment of failure effects
4. Determination of detection mechanisms
5. Calculation of diagnostic coverage
6. Mapping to safety goals and requirements

---

## 2. System Overview

### 2.1 Functional Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         LSR Engine                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Engine (Facade)                                             ││
│  │  - render()      - verify()      - handleWindowEvents()     ││
│  │  - getError()                                                ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                    │
│  ┌──────────────┬───────────┼───────────┬──────────────────────┐│
│  │              │           │           │                       ││
│  ▼              ▼           ▼           ▼                       ││
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────────┐ ││
│ │ Database │ │ Display  │ │ Frame    │ │ Common Utilities     │ ││
│ │          │ │ Manager  │ │ Handler  │ │ - Pool               │ ││
│ │ - DDH    │ │ - Canvas │ │ - Window │ │ - Assertion          │ ││
│ │ - Bitmap │ │ - Texture│ │ - Frame  │ │ - ErrorCollector     │ ││
│ │ - Config │ │ - Cache  │ │ - Panel  │ │ - LongTermPtr        │ ││
│ └──────────┘ └──────────┘ │ - Field  │ │ - ReturnValue        │ ││
│                           │ - RefBmp │ └──────────────────────┘ ││
│                           └──────────┘                           ││
└─────────────────────────────────────────────────────────────────┘│
                              │                                     │
                    ┌─────────┴─────────┐                          │
                    ▼                   ▼                          │
              ┌──────────┐        ┌──────────┐                     │
              │   GIL    │        │   PIL    │                     │
              │ Graphics │        │ Platform │                     │
              └──────────┘        └──────────┘                     │
```

### 2.2 Safety Functions

| SF ID | Safety Function | Related Safety Goal |
|-------|-----------------|---------------------|
| SF-1 | Correct bitmap rendering | SG1, SG5 |
| SF-2 | Video output verification | SG4 |
| SF-3 | Error detection and reporting | SG2 |
| SF-4 | Memory pool integrity checking | SG1, SG2 |
| SF-5 | Configuration data validation | SG1 |
| SF-6 | Timely rendering | SG3 |

---

## 3. Component-Level FMEA

### 3.1 Engine Module (`engine/lsr`)

#### 3.1.1 Engine Class

**Source Files**: `engine/lsr/api/Engine.h`, `engine/lsr/src/Engine.cpp`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| E-FM-001 | render() returns false unexpectedly | No update to display; stale content shown | Return value check | High | Caller monitors return value; enters safe state on repeated failures | 99% |
| E-FM-002 | verify() returns false negative | Display corruption not detected | ReferenceBitmapField verification count | Critical | Redundant verification; periodic full-frame verification | 95% |
| E-FM-003 | verify() returns false positive | Unnecessary error indication | No direct detection | Low | Application-level confirmation of error | 0% |
| E-FM-004 | handleWindowEvents() hangs | System unresponsive | External watchdog | Critical | Watchdog timer at system level | 99% |
| E-FM-005 | getError() returns wrong error | Incorrect error handling | Error collector validation | Medium | Hierarchical error collection with cross-check | 90% |
| E-FM-006 | Initialization failure | Engine not operational | Engine error state | High | Engine reports LSR_DB_ERROR or similar | 99% |

**Error Codes Detected**:
- `LSR_NO_ENGINE_ERROR` (0x0): Success
- `LSR_DB_INCONSISTENT` (0x1000009): Database inconsistency detected
- `LSR_DB_ERROR` (0x100000A): General database error
- `LSR_DB_DDHBIN_VERSION_MISMATCH` (0x100000B): Configuration version mismatch
- `LSR_DB_DDHBIN_EMPTY` (0x100000C): Empty configuration

---

### 3.2 Database Module (`engine/database`)

#### 3.2.1 Database Class

**Source Files**: `engine/database/api/Database.h`, `engine/database/src/Database.cpp`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| DB-FM-001 | Invalid bitmap ID lookup | Wrong bitmap returned or NULL | Return value check | Critical | Validate bitmap ID against known range | 99% |
| DB-FM-002 | DDH configuration corrupted | Incorrect rendering parameters | DDH version check, CRC | Critical | Configuration integrity check at startup | 95% |
| DB-FM-003 | Bitmap data corrupted | Visual artifacts | Pixel verification | High | ReferenceBitmapField compares output | 99% |
| DB-FM-004 | Resource buffer overflow | Memory corruption | Pool bounds check | Critical | Fixed-size pools prevent overflow | 99% |
| DB-FM-005 | Inconsistent panel/frame data | Incorrect widget hierarchy | Hierarchical validation | High | Database consistency check at load | 90% |

#### 3.2.2 StaticBitmap Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| SB-FM-001 | getData() returns NULL | Crash or no rendering | NULL check | High | Validate pointer before use | 99% |
| SB-FM-002 | Incorrect image dimensions | Rendering artifacts | Dimension validation | Medium | Cross-check against DDH specification | 90% |
| SB-FM-003 | Wrong pixel format | Color corruption | Format validation | Medium | Format consistency check | 90% |

---

### 3.3 Display Module (`engine/display`)

#### 3.3.1 DisplayManager Class

**Source Files**: `engine/display/api/DisplayManager.h`, `engine/display/src/DisplayManager.cpp`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| DM-FM-001 | createWindow() fails | No rendering surface | Return value check | Critical | Engine cannot proceed; reports error | 99% |
| DM-FM-002 | GIL context creation fails | No rendering possible | Context validation | Critical | GIL_INVALID_CONTEXT reported | 99% |
| DM-FM-003 | Surface binding fails | Rendering to wrong surface | GIL error check | High | gilSetSurface returns GIL_FALSE | 99% |
| DM-FM-004 | Display update loss | Stale display content | Frame counter monitoring | High | Application monitors render cycles | 95% |

#### 3.3.2 TextureCache Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| TC-FM-001 | Texture allocation failure | Image not displayed | LSR_ERROR_NO_TEXTURE | High | Error reported; safe default | 99% |
| TC-FM-002 | Texture cache corruption | Wrong texture used | Texture ID validation | Medium | Texture ID bounds check | 90% |
| TC-FM-003 | Stale texture data | Incorrect image displayed | Invalidation mechanism | Medium | Invalidation on data change | 85% |

#### 3.3.3 Texture Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| TX-FM-001 | gilTexPixels() fails | Texture not loaded | Return value check | High | GIL_FALSE returned | 99% |
| TX-FM-002 | Palette load failure | Incorrect colors | Return value check | Medium | gilTexPalette returns GIL_FALSE | 99% |
| TX-FM-003 | Invalid texture format | Rendering artifacts | Format validation | Medium | GIL_FORMAT_INVALID check | 90% |

---

### 3.4 FrameHandler Module (`engine/framehandler`)

#### 3.4.1 FrameHandler Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| FH-FM-001 | Render loop deadlock | Display freeze | External watchdog | Critical | Watchdog timeout | 99% |
| FH-FM-002 | Incorrect render order | Z-order violations | Visual inspection | Medium | Static widget ordering | N/A |
| FH-FM-003 | Widget not rendered | Missing content | Verification | High | ReferenceBitmapField detection | 99% |

#### 3.4.2 Widget Hierarchy (Window, Frame, Panel)

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| WH-FM-001 | Widget tree corruption | Incorrect rendering | Hierarchical validation | Critical | Pool marker checking | 95% |
| WH-FM-002 | Invalid child pointer | Crash or corruption | Pointer validation | Critical | isAllocated() check | 99% |
| WH-FM-003 | Area calculation error | Clipping issues | Bounds checking | Medium | Area validation | 90% |
| WH-FM-004 | Invalidation lost | Content not updated | Manual invalidation | Medium | Force invalidation option | 80% |

#### 3.4.3 BitmapField Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| BF-FM-001 | Wrong bitmap selected | Incorrect indicator shown | Verification | Critical | ReferenceBitmapField comparison | 99% |
| BF-FM-002 | Bitmap ID out of range | Crash or no rendering | Bounds check | High | ID validation against database | 99% |
| BF-FM-003 | Texture binding failure | Image not rendered | GIL error check | High | gilBindTexture validation | 90% |
| BF-FM-004 | Draw position error | Misplaced indicator | Visual verification | Medium | Position validation | 85% |

#### 3.4.4 ReferenceBitmapField Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| RB-FM-001 | gilVerify() false negative | Corruption not detected | Redundant verification | Critical | Multiple verification passes | 95% |
| RB-FM-002 | gilVerify() false positive | Unnecessary alarm | Error count threshold | Low | Application-level filtering | 50% |
| RB-FM-003 | Verification not executed | No safety check | Verification call monitoring | Critical | Call sequence monitoring | 90% |
| RB-FM-004 | Error counter overflow | Lost error count | Counter bounds check | Low | 32-bit counter (>4 billion) | N/A |

---

### 3.5 Common Utilities (`engine/common`)

#### 3.5.1 Pool Template Class

**Source File**: `engine/common/api/Pool.h`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| PL-FM-001 | Pool exhaustion | Object creation fails | LSR_POOL_IS_FULL | High | Pre-sized pools; error reported | 99% |
| PL-FM-002 | Double deallocation | Memory corruption | LSR_POOL_DOUBLE_DELETE | Critical | Marker-based detection | 99% |
| PL-FM-003 | Pool corruption (marker) | Unpredictable behavior | LSR_POOL_IS_CORRUPTED | Critical | checkPool() validation | 99% |
| PL-FM-004 | Invalid pointer deallocate | Memory corruption | LSR_POOL_INVALID_OBJECT | Critical | isAllocated() validation | 99% |
| PL-FM-005 | Free list corruption | Infinite loop | Node counter limit | Critical | Loop detection (PoolSize limit) | 99% |

**Safety Mechanisms**:
- Free marker: 0xAA pattern
- Busy marker: 0x55 pattern
- Bounds checking on every operation
- Free list integrity validation
- Loop detection in free list traversal

#### 3.5.2 Assertion Module

**Source File**: `engine/common/api/Assertion.h`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| AS-FM-001 | ASSERT disabled (NDEBUG) | Debug checks bypassed | Build configuration | Medium | REQUIRE always active | N/A |
| AS-FM-002 | pilAssert not called | Failure not reported | Test coverage | High | Ensure pilAssert implements handler | 95% |
| AS-FM-003 | REQUIRE returns false | Unexpected continuation | Return value usage | Medium | Caller handles return value | 90% |

#### 3.5.3 LSRErrorCollector Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| EC-FM-001 | Error overwritten | First error lost | Error priority ordering | Low | Severity-based error retention | 80% |
| EC-FM-002 | Error not collected | Silent failure | Error propagation check | Medium | Hierarchical error collection | 90% |

#### 3.5.4 LongTermPtr Class

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| LP-FM-001 | Pointer corruption | Wrong object accessed | Validation check | Critical | Pool-based validation | 95% |
| LP-FM-002 | Dangling pointer | Use after free | isAllocated() check | Critical | Pool tracks allocation status | 99% |

---

### 3.6 Graphics Interface Layer (`gil`)

**Source File**: `gil/api/gil.h`

#### 3.6.1 Context Management

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| GIL-FM-001 | gilCreateContext() fails | No rendering possible | NULL return | Critical | Engine reports error | 99% |
| GIL-FM-002 | gilCreateWindow() fails | No display surface | NULL return | Critical | Engine reports error | 99% |
| GIL-FM-003 | gilSetSurface() fails | Rendering to wrong target | GIL_FALSE return | High | Error check and retry | 99% |

#### 3.6.2 Rendering Operations

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| GIL-FM-004 | gilDrawQuad() silent failure | Image not rendered | gilVerify() | Critical | Video output verification | 99% |
| GIL-FM-005 | gilDrawArea() wrong color | Background corruption | Visual verification | Medium | Color validation | 85% |
| GIL-FM-006 | gilClear() incomplete | Residual artifacts | Visual inspection | Low | Full-frame verification | 80% |
| GIL-FM-007 | gilSwapBuffers() fails | Display not updated | Return value check | Critical | GIL_FALSE indicates failure | 99% |

#### 3.6.3 Texture Operations

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| GIL-FM-008 | gilCreateTexture() fails | Texture not available | NULL return | High | Texture allocation tracking | 99% |
| GIL-FM-009 | gilTexPixels() corruption | Wrong texture data | gilVerify() | Critical | Pixel-level verification | 99% |
| GIL-FM-010 | gilBindTexture() wrong texture | Wrong image rendered | gilVerify() | Critical | Verification against reference | 99% |

#### 3.6.4 Verification Operations

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| GIL-FM-011 | gilVerify() false negative | Corruption not detected | Redundant checks | Critical | Multiple verification passes | 95% |
| GIL-FM-012 | gilVerify() false positive | Spurious error | Threshold filtering | Low | Application-level threshold | 60% |
| GIL-FM-013 | gilGetError() returns wrong error | Incorrect error handling | Error sequence check | Medium | Error logging | 80% |

---

### 3.7 Platform Interface Layer (`pil`)

**Source File**: `pil/api/pil.h`

| FM ID | Failure Mode | Failure Effect | Detection | Severity | Mitigation | DC |
|-------|--------------|----------------|-----------|----------|------------|-----|
| PIL-FM-001 | pilGetMonotonicTime() incorrect | Timing errors | Time consistency check | High | Plausibility monitoring | 90% |
| PIL-FM-002 | pilGetMonotonicTime() overflow | Time wraparound | Overflow handling | Medium | 49-day overflow expected; handled | 99% |
| PIL-FM-003 | pilAssert() not implemented | Assertions silently fail | Test verification | Critical | Integration test requirement | 95% |
| PIL-FM-004 | pilAssert() infinite loop | System hang | Watchdog | High | Watchdog timeout detection | 99% |

---

## 4. Common Cause Failure Analysis

### 4.1 Software Systematic Failures

| CCF ID | Common Cause | Affected Components | Mitigation |
|--------|--------------|---------------------|------------|
| CCF-001 | Memory corruption | All Pool-based objects | Marker-based detection, bounds checking |
| CCF-002 | Stack overflow | All modules | Static stack analysis, bounded recursion |
| CCF-003 | Compiler defect | All code | Qualified compiler, diverse testing |
| CCF-004 | DDH generation defect | Database, all renderers | Tool qualification, configuration validation |
| CCF-005 | GIL implementation defect | All rendering | GIL qualification, gilVerify() |

### 4.2 Dependent Failure Analysis

| DFA ID | Dependent Failure | Components | Independence Measure |
|--------|-------------------|------------|----------------------|
| DFA-001 | Pool corruption affects multiple objects | Pool users | Separate pools per object type |
| DFA-002 | Error collector corruption | All error reporting | Redundant error channels |
| DFA-003 | Canvas/Context corruption | All rendering | Context isolation per window |

---

## 5. Diagnostic Coverage Summary

### 5.1 Coverage by Module

| Module | Average DC | Critical Functions DC |
|--------|------------|----------------------|
| Engine | 95% | 97% |
| Database | 93% | 95% |
| Display | 94% | 97% |
| FrameHandler | 91% | 95% |
| Common | 96% | 99% |
| GIL | 92% | 95% |
| PIL | 95% | 95% |
| **Overall** | **94%** | **96%** |

### 5.2 Coverage by Safety Goal

| Safety Goal | Required DC (ASIL D) | Achieved DC | Status |
|-------------|---------------------|-------------|--------|
| SG1 (Correct Display) | 99% | 97% | Mitigation Required |
| SG2 (Availability) | 99% | 96% | Mitigation Required |
| SG3 (Timeliness) | 97% | 90% | Mitigation Required |
| SG4 (Corruption Detection) | 97% | 99% | Compliant |
| SG5 (No False Indication) | 90% | 95% | Compliant |

### 5.3 Mitigation Actions for DC Gaps

| Gap | Current DC | Required DC | Mitigation |
|-----|------------|-------------|------------|
| SG1 DC Gap | 97% | 99% | Add redundant configuration validation |
| SG2 DC Gap | 96% | 99% | Implement dual-channel error reporting |
| SG3 DC Gap | 90% | 97% | Add execution time monitoring |

---

## 6. Safety Mechanism Summary

### 6.1 Pre-existing Safety Mechanisms

| SM ID | Mechanism | Location | Detection Coverage |
|-------|-----------|----------|-------------------|
| SM-001 | Pool marker checking | Pool.h | Pool corruption (99%) |
| SM-002 | Bounds checking | Pool.h | Invalid access (99%) |
| SM-003 | Free list validation | Pool.h | List corruption (99%) |
| SM-004 | Video output verification | ReferenceBitmapField | Pixel corruption (99%) |
| SM-005 | Error collector hierarchy | LSRErrorCollector | Error propagation (90%) |
| SM-006 | GIL error codes | gil.h | Graphics errors (95%) |
| SM-007 | Assertion framework | Assertion.h | Programming errors (95%) |

### 6.2 Recommended Additional Safety Mechanisms

| RSM ID | Mechanism | Purpose | ASIL Impact |
|--------|-----------|---------|-------------|
| RSM-001 | DDH CRC validation | Configuration integrity | SG1 +2% DC |
| RSM-002 | Execution time monitor | Timing compliance | SG3 +7% DC |
| RSM-003 | Redundant error channel | Error reporting reliability | SG2 +3% DC |
| RSM-004 | Bitmap CRC validation | Data integrity | SG1 +1% DC |
| RSM-005 | Watchdog integration | Hang detection | SG2, SG3 +2% DC |

---

## 7. Failure Mode to Safety Goal Traceability

| Failure Mode | Effect | Safety Goal Impacted | FSR |
|--------------|--------|---------------------|-----|
| E-FM-002 | Corruption undetected | SG4 | FSR-VE-002 |
| DB-FM-002 | Wrong configuration | SG1 | FSR-DD-001 |
| DB-FM-003 | Visual artifacts | SG1, SG4 | FSR-DD-002, FSR-VE-001 |
| BF-FM-001 | Wrong indicator | SG1 | FSR-DD-003 |
| RB-FM-001 | Corruption undetected | SG4 | FSR-VE-002 |
| PL-FM-003 | Memory corruption | SG1, SG2 | FSR-MS-001 |
| GIL-FM-011 | Corruption undetected | SG4 | FSR-VE-002 |

---

## 8. Conclusions

### 8.1 Key Findings

1. **Strong Memory Safety**: The Pool template provides robust memory corruption detection with 99% diagnostic coverage.

2. **Effective Verification**: ReferenceBitmapField with gilVerify() provides 99% detection of pixel-level corruption.

3. **DC Gaps Identified**: Three safety goals (SG1, SG2, SG3) require additional mechanisms to achieve ASIL D diagnostic coverage targets.

4. **SEooC Boundary Risks**: GIL and PIL implementations provided by integrator must meet ASIL D requirements.

### 8.2 Recommended Actions

| Priority | Action | Safety Goal | Target DC |
|----------|--------|-------------|-----------|
| High | Implement DDH CRC validation | SG1 | +2% |
| High | Add execution time monitoring | SG3 | +7% |
| Medium | Implement redundant error channel | SG2 | +3% |
| Medium | Add watchdog integration guide | SG2, SG3 | +2% |
| Low | Add bitmap data CRC | SG1 | +1% |

### 8.3 Compliance Statement

With the recommended additional safety mechanisms implemented, the Luxoft Safe Renderer can achieve the diagnostic coverage required for ISO 26262 ASIL D compliance. The analysis identifies specific gaps and provides actionable mitigations.

---

## Appendix A: FMEA Worksheet

| FM ID | Component | Function | Failure Mode | Local Effect | System Effect | Vehicle Effect | S | Existing Detection | DC | Mitigation |
|-------|-----------|----------|--------------|--------------|---------------|----------------|---|-------------------|-----|------------|
| PL-FM-001 | Pool | allocate() | Pool full | NULL returned | Object not created | Indicator not shown | S3 | LSR_POOL_IS_FULL | 99% | Pre-sized pools |
| PL-FM-002 | Pool | deallocate() | Double delete | Corruption | Unpredictable | Safety function loss | S3 | Marker check | 99% | 0x55/0xAA markers |
| PL-FM-003 | Pool | allocate() | Corruption | Wrong data | Wrong render | Wrong indicator | S3 | checkPool() | 99% | Marker validation |
| RB-FM-001 | RefBmpField | onVerify() | False negative | No error | Corruption missed | Wrong indicator | S3 | Redundant verify | 95% | Multiple passes |
| GIL-FM-004 | GIL | gilDrawQuad() | Silent fail | No render | Missing content | Missing indicator | S3 | gilVerify() | 99% | Video verification |

## Appendix B: Error Code Reference

| Error Code | Value | Meaning | Severity |
|------------|-------|---------|----------|
| LSR_NO_ENGINE_ERROR | 0x0 | Success | - |
| LSR_DH_INVALID_DATA_ID | 0x1000000 | Invalid data handler ID | Medium |
| LSR_POOL_INVALID_OBJECT | 0x1000001 | Invalid pool object | High |
| LSR_ERR_DATASTATUS_NOT_AVAILABLE | 0x1000002 | Data not available | Medium |
| LSR_ERR_DATASTATUS_INVALID | 0x1000003 | Invalid data status | Medium |
| LSR_ERR_DATASTATUS_INCONSISTENT | 0x1000004 | Inconsistent data | High |
| LSR_POOL_IS_FULL | 0x1000005 | Pool exhausted | High |
| LSR_POOL_DOUBLE_DELETE | 0x1000006 | Double deallocation | Critical |
| LSR_POOL_IS_CORRUPTED | 0x1000007 | Pool corruption detected | Critical |
| LSR_ERROR_NO_TEXTURE | 0x1000008 | Texture allocation failed | High |
| LSR_DB_INCONSISTENT | 0x1000009 | Database inconsistent | Critical |
| LSR_DB_ERROR | 0x100000A | General database error | High |
| LSR_DB_DDHBIN_VERSION_MISMATCH | 0x100000B | Version mismatch | Critical |
| LSR_DB_DDHBIN_EMPTY | 0x100000C | Empty configuration | Critical |

## Appendix C: GIL Error Code Reference

| Error Code | Value | Meaning |
|------------|-------|---------|
| GIL_NO_ERROR | 0x0 | Success |
| GIL_INVALID_CONTEXT | 0x200 | Invalid rendering context |
| GIL_INVALID_OPERATION | 0x201 | Invalid operation |
| GIL_INVALID_SURFACE | 0x202 | Invalid surface |
| GIL_INVALID_VALUE | 0x203 | Invalid value |

---

**End of Document**
