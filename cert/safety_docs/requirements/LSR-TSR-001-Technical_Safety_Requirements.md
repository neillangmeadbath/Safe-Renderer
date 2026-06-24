# LSR-TSR-001: Technical Safety Requirements

| Document ID | LSR-TSR-001 |
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
| LSR-SAD-001 | Software Architecture Description |
| LSR-DS-001 | Design Specification |

---

## 1. Introduction

### 1.1 Purpose

This document specifies the Technical Safety Requirements (TSR) for the Luxoft Safe Renderer. TSRs are derived from the Functional Safety Requirements (FSR) and provide implementation-level specifications that can be directly verified through code review, testing, and analysis.

### 1.2 Requirements Notation

**TSR-XX-NNN**: Technical Safety Requirement
- XX: Category code matching FSR category
- NNN: Sequential number

---

## 2. Technical Safety Requirements

### 2.1 Data/Display Requirements (TSR-DD)

#### TSR-DD-001: DDH Magic Number Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-001 |
| **Description** | The Database class shall verify the DDH magic number at initialization and return LSR_DB_ERROR if the magic number is invalid. |
| **ASIL** | D |
| **Derived From** | FSR-DD-001 |
| **Implementation** | `Database::Database()` constructor |
| **Verification** | Unit test with invalid magic number |

#### TSR-DD-002: DDH Structure Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-002 |
| **Description** | The Database class shall validate that all DDH structure pointers are non-NULL and within valid memory ranges before use. |
| **ASIL** | D |
| **Derived From** | FSR-DD-001 |
| **Implementation** | `Database` member access methods |
| **Verification** | Unit test with NULL DDH pointers |

#### TSR-DD-003: DDH Version Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-003 |
| **Description** | The Database class shall compare the DDH binary version against DDHBIN_VERSION and return LSR_DB_DDHBIN_VERSION_MISMATCH if they differ. |
| **ASIL** | D |
| **Derived From** | FSR-DD-002 |
| **Implementation** | `Database::Database()` |
| **Verification** | Unit test with mismatched versions |

#### TSR-DD-004: Bitmap Data NULL Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-004 |
| **Description** | The StaticBitmap class shall validate that bitmap data pointer is non-NULL before returning it via getData(). |
| **ASIL** | D |
| **Derived From** | FSR-DD-003 |
| **Implementation** | `StaticBitmap::getData()` |
| **Verification** | Unit test with NULL bitmap data |

#### TSR-DD-005: Bitmap Dimension Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-005 |
| **Description** | The StaticBitmap class shall validate that bitmap width and height are greater than zero and within maximum supported dimensions. |
| **ASIL** | D |
| **Derived From** | FSR-DD-003 |
| **Implementation** | `StaticBitmap::getWidth()`, `StaticBitmap::getHeight()` |
| **Verification** | Unit test with zero/invalid dimensions |

#### TSR-DD-006: Bitmap ID Range Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-006 |
| **Description** | The Database class shall validate bitmap IDs against the configured maximum count and return NULL for out-of-range IDs. |
| **ASIL** | D |
| **Derived From** | FSR-DD-004 |
| **Implementation** | `Database::getBitmap()` |
| **Verification** | Unit test with out-of-range bitmap ID |

#### TSR-DD-007: Render Position Calculation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-007 |
| **Description** | The BitmapField class shall calculate render position from Area coordinates using integer arithmetic without overflow. |
| **ASIL** | D |
| **Derived From** | FSR-DD-005 |
| **Implementation** | `BitmapField::onDraw()` |
| **Verification** | Unit test with boundary positions |

#### TSR-DD-008: Texture Coordinate Calculation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-DD-008 |
| **Description** | The Canvas class shall calculate texture UV coordinates correctly to ensure 1:1 pixel mapping for unscaled rendering. |
| **ASIL** | D |
| **Derived From** | FSR-DD-005 |
| **Implementation** | `Canvas::drawBitmap()` |
| **Verification** | Pixel-level verification testing |

---

### 2.2 Availability Requirements (TSR-AV)

#### TSR-AV-001: Render Return Value

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-001 |
| **Description** | The Engine::render() method shall return true on successful render completion and false on any failure. |
| **ASIL** | D |
| **Derived From** | FSR-AV-001 |
| **Implementation** | `Engine::render()` |
| **Verification** | Unit test render success/failure cases |

#### TSR-AV-002: Frame Handler Render Completion

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-002 |
| **Description** | The FrameHandler::render() method shall traverse the complete widget tree and return true only if all widgets rendered successfully. |
| **ASIL** | D |
| **Derived From** | FSR-AV-001 |
| **Implementation** | `FrameHandler::render()` |
| **Verification** | Unit test with partial render failures |

#### TSR-AV-003: Error Aggregation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-003 |
| **Description** | The Engine::getError() method shall return the highest-severity error from Database, DisplayManager, and FrameHandler components. |
| **ASIL** | D |
| **Derived From** | FSR-AV-002 |
| **Implementation** | `Engine::getError()` |
| **Verification** | Unit test error aggregation |

#### TSR-AV-004: Component Error Collection

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-004 |
| **Description** | Each component (Database, DisplayManager, FrameHandler) shall maintain its current error state accessible via a getError() method. |
| **ASIL** | D |
| **Derived From** | FSR-AV-002 |
| **Implementation** | Component getError() methods |
| **Verification** | Unit test per-component error reporting |

#### TSR-AV-005: Error State Persistence

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-005 |
| **Description** | Once a critical error (LSR_POOL_IS_CORRUPTED, LSR_DB_ERROR, LSR_DB_INCONSISTENT) is recorded, the Engine shall retain this error state until explicitly reset. |
| **ASIL** | D |
| **Derived From** | FSR-AV-003 |
| **Implementation** | `Engine::m_error` state management |
| **Verification** | Unit test error persistence |

#### TSR-AV-006: Widget Pointer Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AV-006 |
| **Description** | The WidgetChildren container shall validate child pointers using Pool::isAllocated() before dereferencing. |
| **ASIL** | D |
| **Derived From** | FSR-AV-004 |
| **Implementation** | `WidgetChildren::operator[]` |
| **Verification** | Unit test with invalid child pointers |

---

### 2.3 Timing Requirements (TSR-TI)

#### TSR-TI-001: Bounded Render Loop

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-TI-001 |
| **Description** | The render loop shall have bounded execution time determined by the number of widgets (O(n) where n = widget count) without unbounded loops. |
| **ASIL** | C |
| **Derived From** | FSR-TI-001 |
| **Implementation** | `FrameHandler::render()` |
| **Verification** | Static analysis; timing measurement |

#### TSR-TI-002: Time Query Interface

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-TI-002 |
| **Description** | The Timer class shall provide current time via pilGetMonotonicTime() for timing measurements by the integration layer. |
| **ASIL** | C |
| **Derived From** | FSR-TI-002 |
| **Implementation** | `Timer` class |
| **Verification** | Unit test timing interface |

#### TSR-TI-003: Minimum Frame Rate Support

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-TI-003 |
| **Description** | The GIL swap buffer operation shall complete within 100ms to support minimum 10 Hz update rate. |
| **ASIL** | C |
| **Derived From** | FSR-TI-003 |
| **Implementation** | `gilSwapBuffers()` |
| **Verification** | Performance testing |

---

### 2.4 Verification Requirements (TSR-VE)

#### TSR-VE-001: Pixel Verification Call

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-VE-001 |
| **Description** | The ReferenceBitmapField::onVerify() method shall call gilVerify() with correct coordinates and texture reference. |
| **ASIL** | C |
| **Derived From** | FSR-VE-001 |
| **Implementation** | `ReferenceBitmapField::onVerify()` |
| **Verification** | Unit test with mock GIL |

#### TSR-VE-002: Verification Comparison

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-VE-002 |
| **Description** | The gilVerify() function shall compare each pixel in the specified area against the reference texture and return GIL_FALSE if any pixel differs. |
| **ASIL** | C |
| **Derived From** | FSR-VE-001 |
| **Implementation** | `gilVerify()` in GIL |
| **Verification** | Pixel-level fault injection testing |

#### TSR-VE-003: Full Area Coverage

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-VE-003 |
| **Description** | The verification shall check every pixel within the ReferenceBitmapField area bounds (100% pixel coverage). |
| **ASIL** | C |
| **Derived From** | FSR-VE-002 |
| **Implementation** | `gilVerify()` loop |
| **Verification** | Coverage analysis of verification |

#### TSR-VE-004: Error Counter Increment

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-VE-004 |
| **Description** | The ReferenceBitmapField shall increment m_verificationErrors by 1 for each failed verification (gilVerify returns GIL_FALSE). |
| **ASIL** | C |
| **Derived From** | FSR-VE-003 |
| **Implementation** | `ReferenceBitmapField::onVerify()` |
| **Verification** | Unit test error counter |

#### TSR-VE-005: Visibility Flag Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-VE-005 |
| **Description** | The ReferenceBitmapField::onVerify() shall skip verification and return true if the visible flag is false. |
| **ASIL** | C |
| **Derived From** | FSR-VE-004 |
| **Implementation** | `ReferenceBitmapField::onVerify()` |
| **Verification** | Unit test visibility control |

---

### 2.5 Memory Safety Requirements (TSR-MS)

#### TSR-MS-001: Pool Marker Pattern

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-001 |
| **Description** | The Pool class shall use marker bytes 0xAA for free nodes and 0x55 for allocated nodes to detect corruption. |
| **ASIL** | D |
| **Derived From** | FSR-MS-001 |
| **Implementation** | `Pool::MARKER_FREE_CHAR`, `Pool::MARKER_BUSY_CHAR` |
| **Verification** | Unit test marker detection |

#### TSR-MS-002: Pool Integrity Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-002 |
| **Description** | The Pool::checkPool() method shall verify: (1) standard markers are intact, (2) all nodes have valid markers, (3) free list has no loops. |
| **ASIL** | D |
| **Derived From** | FSR-MS-001 |
| **Implementation** | `Pool::checkPool()` |
| **Verification** | Unit test with various corruptions |

#### TSR-MS-003: Double Delete Detection

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-003 |
| **Description** | The Pool::deallocate() method shall return LSR_POOL_DOUBLE_DELETE if the object's marker indicates it is already free (0xAA pattern). |
| **ASIL** | D |
| **Derived From** | FSR-MS-002 |
| **Implementation** | `Pool::deallocate()` |
| **Verification** | Unit test double deallocation |

#### TSR-MS-004: Pool Full Detection

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-004 |
| **Description** | The Pool::allocate() method shall return NULL and set error to LSR_POOL_IS_FULL when m_pFreeList is NULL. |
| **ASIL** | D |
| **Derived From** | FSR-MS-003 |
| **Implementation** | `Pool::allocate()` |
| **Verification** | Unit test pool exhaustion |

#### TSR-MS-005: Pointer Bounds Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-005 |
| **Description** | The Pool::isAllocated() method shall verify: (1) pointer is within storage bounds, (2) pointer is node-aligned, (3) marker is valid. |
| **ASIL** | D |
| **Derived From** | FSR-MS-004 |
| **Implementation** | `Pool::isAllocated()` |
| **Verification** | Unit test with various invalid pointers |

#### TSR-MS-006: Static Pool Sizing

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-MS-006 |
| **Description** | All Pool template instantiations shall use compile-time fixed sizes; no runtime pool size changes shall be permitted. |
| **ASIL** | D |
| **Derived From** | FSR-MS-005 |
| **Implementation** | `Pool<T, PoolSize>` template |
| **Verification** | Static analysis; code review |

---

### 2.6 Error Handling Requirements (TSR-ER)

#### TSR-ER-001: Error Collector Hierarchy

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-ER-001 |
| **Description** | The LSRErrorCollector class shall compare errors by numeric value and retain the highest value (most severe). |
| **ASIL** | D |
| **Derived From** | FSR-ER-001 |
| **Implementation** | `LSRErrorCollector::setError()` |
| **Verification** | Unit test error ordering |

#### TSR-ER-002: Error Domain Encoding

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-ER-002 |
| **Description** | Error codes shall use offset 0x1000000 to distinguish engine errors from success (0) and allow domain identification. |
| **ASIL** | D |
| **Derived From** | FSR-ER-002 |
| **Implementation** | `LSREngineError` enum |
| **Verification** | Code review; static analysis |

#### TSR-ER-003: Error Code Uniqueness

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-ER-003 |
| **Description** | Each distinct error condition shall have a unique error code in the LSREngineError enumeration. |
| **ASIL** | D |
| **Derived From** | FSR-ER-002 |
| **Implementation** | `LSREngineError` enum |
| **Verification** | Code review; enum analysis |

#### TSR-ER-004: Assertion Callback

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-ER-004 |
| **Description** | The executeAssert() function shall call pilAssert() with file name, line number, and assertion message. |
| **ASIL** | D |
| **Derived From** | FSR-ER-003 |
| **Implementation** | `lsr::impl::executeAssert()` |
| **Verification** | Unit test assertion invocation |

---

### 2.7 Initialization Requirements (TSR-IN)

#### TSR-IN-001: Database Initialization Sequence

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-IN-001 |
| **Description** | The Database class constructor shall: (1) validate DDH, (2) load configuration, (3) set error state before returning. |
| **ASIL** | D |
| **Derived From** | FSR-IN-001 |
| **Implementation** | `Database::Database()` |
| **Verification** | Unit test initialization sequence |

#### TSR-IN-002: Display Manager Initialization

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-IN-002 |
| **Description** | The DisplayManager class shall initialize GIL context and verify successful creation before accepting render requests. |
| **ASIL** | D |
| **Derived From** | FSR-IN-001 |
| **Implementation** | `DisplayManager` constructor |
| **Verification** | Unit test with GIL init failures |

#### TSR-IN-003: Initialization Error Blocking

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-IN-003 |
| **Description** | If initialization fails (m_error != LSR_NO_ENGINE_ERROR), Engine::render() shall return false without performing rendering. |
| **ASIL** | D |
| **Derived From** | FSR-IN-002 |
| **Implementation** | `Engine::render()` |
| **Verification** | Unit test render after init failure |

---

### 2.8 False Indication Requirements (TSR-FI)

#### TSR-FI-001: Data Status Enumeration

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-FI-001 |
| **Description** | Data status shall be represented using DataStatus enumeration with values: VALID, NOT_AVAILABLE, INVALID, INCONSISTENT. |
| **ASIL** | A |
| **Derived From** | FSR-FI-001 |
| **Implementation** | `DataStatus` enum |
| **Verification** | Code review |

#### TSR-FI-002: Status Check Before Render

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-FI-002 |
| **Description** | The BitmapField class shall check data status and skip rendering if status is NOT_AVAILABLE or INVALID. |
| **ASIL** | A |
| **Derived From** | FSR-FI-001 |
| **Implementation** | `BitmapField::onDraw()` |
| **Verification** | Unit test with invalid status |

#### TSR-FI-003: Not Available Handling

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-FI-003 |
| **Description** | When data status is NOT_AVAILABLE, the Field shall set its visible flag to false, preventing rendering. |
| **ASIL** | A |
| **Derived From** | FSR-FI-002 |
| **Implementation** | `Field::update()` |
| **Verification** | Unit test visibility on NOT_AVAILABLE |

---

## 3. Requirements Traceability

### 3.1 FSR to TSR Traceability Matrix

| FSR ID | TSR IDs |
|--------|---------|
| FSR-DD-001 | TSR-DD-001, TSR-DD-002 |
| FSR-DD-002 | TSR-DD-003 |
| FSR-DD-003 | TSR-DD-004, TSR-DD-005 |
| FSR-DD-004 | TSR-DD-006 |
| FSR-DD-005 | TSR-DD-007, TSR-DD-008 |
| FSR-AV-001 | TSR-AV-001, TSR-AV-002 |
| FSR-AV-002 | TSR-AV-003, TSR-AV-004 |
| FSR-AV-003 | TSR-AV-005 |
| FSR-AV-004 | TSR-AV-006 |
| FSR-TI-001 | TSR-TI-001 |
| FSR-TI-002 | TSR-TI-002 |
| FSR-TI-003 | TSR-TI-003 |
| FSR-VE-001 | TSR-VE-001, TSR-VE-002 |
| FSR-VE-002 | TSR-VE-003 |
| FSR-VE-003 | TSR-VE-004 |
| FSR-VE-004 | TSR-VE-005 |
| FSR-MS-001 | TSR-MS-001, TSR-MS-002 |
| FSR-MS-002 | TSR-MS-003 |
| FSR-MS-003 | TSR-MS-004 |
| FSR-MS-004 | TSR-MS-005 |
| FSR-MS-005 | TSR-MS-006 |
| FSR-ER-001 | TSR-ER-001, TSR-ER-002 |
| FSR-ER-002 | TSR-ER-002, TSR-ER-003 |
| FSR-ER-003 | TSR-ER-004 |
| FSR-IN-001 | TSR-IN-001, TSR-IN-002 |
| FSR-IN-002 | TSR-IN-003 |
| FSR-FI-001 | TSR-FI-001, TSR-FI-002 |
| FSR-FI-002 | TSR-FI-003 |

### 3.2 TSR to Code Traceability

| TSR ID | Source File | Function/Class |
|--------|-------------|----------------|
| TSR-DD-001 | engine/database/src/Database.cpp | Database::Database() |
| TSR-DD-002 | engine/database/src/Database.cpp | Database member methods |
| TSR-DD-003 | engine/database/src/Database.cpp | Database::Database() |
| TSR-DD-004 | engine/database/api/StaticBitmap.h | StaticBitmap::getData() |
| TSR-DD-005 | engine/database/api/StaticBitmap.h | StaticBitmap::getWidth/Height() |
| TSR-DD-006 | engine/database/src/Database.cpp | Database::getBitmap() |
| TSR-DD-007 | engine/framehandler/src/BitmapField.cpp | BitmapField::onDraw() |
| TSR-DD-008 | engine/display/src/Canvas.cpp | Canvas::drawBitmap() |
| TSR-AV-001 | engine/lsr/src/Engine.cpp | Engine::render() |
| TSR-AV-002 | engine/framehandler/src/FrameHandler.cpp | FrameHandler::render() |
| TSR-AV-003 | engine/lsr/src/Engine.cpp | Engine::getError() |
| TSR-AV-004 | Various | Component getError() methods |
| TSR-AV-005 | engine/lsr/src/Engine.cpp | Engine error state |
| TSR-AV-006 | engine/framehandler/api/WidgetChildren.h | WidgetChildren access |
| TSR-MS-001 | engine/common/api/Pool.h | Pool marker constants |
| TSR-MS-002 | engine/common/api/Pool.h | Pool::checkPool() |
| TSR-MS-003 | engine/common/api/Pool.h | Pool::deallocate() |
| TSR-MS-004 | engine/common/api/Pool.h | Pool::allocate() |
| TSR-MS-005 | engine/common/api/Pool.h | Pool::isAllocated() |
| TSR-MS-006 | engine/common/api/Pool.h | Pool template |
| TSR-VE-001 | engine/framehandler/src/ReferenceBitmapField.cpp | onVerify() |
| TSR-VE-002 | gil/src/*/gil.c | gilVerify() |
| TSR-VE-003 | gil/src/*/gil.c | gilVerify() loop |
| TSR-VE-004 | engine/framehandler/src/ReferenceBitmapField.cpp | onVerify() |
| TSR-VE-005 | engine/framehandler/src/ReferenceBitmapField.cpp | onVerify() |
| TSR-ER-001 | engine/common/api/LSRErrorCollector.h | LSRErrorCollector |
| TSR-ER-002 | engine/common/api/LSREngineError.h | LSREngineError enum |
| TSR-ER-003 | engine/common/api/LSREngineError.h | LSREngineError enum |
| TSR-ER-004 | engine/common/src/Assertion.cpp | executeAssert() |
| TSR-IN-001 | engine/database/src/Database.cpp | Database constructor |
| TSR-IN-002 | engine/display/src/DisplayManager.cpp | DisplayManager |
| TSR-IN-003 | engine/lsr/src/Engine.cpp | Engine::render() |

---

## 4. Verification Requirements

### 4.1 Verification Methods

| Method | Description | Applicable TSRs |
|--------|-------------|-----------------|
| UT | Unit Testing | All TSRs |
| IT | Integration Testing | TSR-AV-*, TSR-IN-* |
| CR | Code Review | TSR-MS-006, TSR-ER-002, TSR-ER-003 |
| SA | Static Analysis | TSR-MS-006, TSR-TI-001 |
| FI | Fault Injection | TSR-MS-*, TSR-VE-* |
| PT | Performance Testing | TSR-TI-001, TSR-TI-003 |

### 4.2 Coverage Requirements (ASIL D)

| Coverage Type | Requirement |
|---------------|-------------|
| Statement Coverage | 100% |
| Branch Coverage | 100% |
| MC/DC Coverage | 100% for safety-critical decisions |

---

## 5. Summary

| Category | TSR Count | ASIL D | ASIL C | ASIL A |
|----------|-----------|--------|--------|--------|
| DD | 8 | 8 | 0 | 0 |
| AV | 6 | 6 | 0 | 0 |
| TI | 3 | 0 | 3 | 0 |
| VE | 5 | 0 | 5 | 0 |
| MS | 6 | 6 | 0 | 0 |
| ER | 4 | 4 | 0 | 0 |
| IN | 3 | 3 | 0 | 0 |
| FI | 3 | 0 | 0 | 3 |
| **Total** | **38** | **27** | **8** | **3** |

---

**End of Document**
