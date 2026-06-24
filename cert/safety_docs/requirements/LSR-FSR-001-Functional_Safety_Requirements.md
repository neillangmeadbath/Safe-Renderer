# LSR-FSR-001: Functional Safety Requirements

| Document ID | LSR-FSR-001 |
|-------------|--------------|
| Version | 1.0 |
| Date | 2026-05-12 |
| Status | Draft |
| Classification | Safety-Critical |
| Standard | ISO 26262:2018 Part 4, Part 6 |
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
| LSR-SAR-001 | Safety Analysis Report (FMEA) |
| LSR-TSR-001 | Technical Safety Requirements |
| ISO 26262:2018 | Road vehicles - Functional safety |

---

## 1. Introduction

### 1.1 Purpose

This document specifies the Functional Safety Requirements (FSR) for the Luxoft Safe Renderer (LSR). These requirements are derived from the Safety Goals defined in LSR-HARA-001 and define the safety functions that must be implemented to achieve ISO 26262 ASIL D compliance.

### 1.2 Scope

This document covers all safety-related functional requirements for:
- Core rendering engine (`engine/lsr`)
- Database management (`engine/database`)
- Display management (`engine/display`)
- Frame handling (`engine/framehandler`)
- Common utilities (`engine/common`)
- External interfaces (GIL, PIL)

### 1.3 Requirements Notation

Requirements are identified as follows:
- **FSR-XX-NNN**: Functional Safety Requirement
  - XX: Category code (see Section 1.4)
  - NNN: Sequential number

**Requirement Attributes**:
| Attribute | Description |
|-----------|-------------|
| ID | Unique requirement identifier |
| Description | Requirement statement |
| ASIL | Assigned safety integrity level |
| Derived From | Parent safety goal(s) |
| FTTI | Fault Tolerant Time Interval |
| Safe State | System state upon violation |
| Verification | Method to verify compliance |

### 1.4 Category Codes

| Code | Category | Description |
|------|----------|-------------|
| DD | Data/Display | Correct display of safety indicators |
| AV | Availability | Availability of safety functions |
| TI | Timing | Timeliness of safety functions |
| VE | Verification | Video output verification |
| MS | Memory Safety | Memory integrity protection |
| ER | Error Handling | Error detection and reporting |
| IN | Initialization | System startup requirements |
| FI | False Indication | Prevention of false displays |

---

## 2. Safety Goals Summary

From LSR-HARA-001:

| SG ID | Safety Goal | ASIL |
|-------|-------------|------|
| SG1 | Correct Display of Safety Indicators | D |
| SG2 | Availability of Safety Indicators | D |
| SG3 | Timeliness of Safety Indicators | C |
| SG4 | Detection of Display Corruption | C |
| SG5 | Avoidance of False Indications | A |

---

## 3. Functional Safety Requirements

### 3.1 Data/Display Requirements (FSR-DD)

#### FSR-DD-001: Configuration Data Validation

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-DD-001 |
| **Description** | The LSR shall validate the integrity of DDH configuration data at system startup before rendering operations commence. |
| **ASIL** | D |
| **Derived From** | SG1 |
| **FTTI** | N/A (startup only) |
| **Safe State** | Engine reports LSR_DB_ERROR; no rendering |
| **Rationale** | Corrupted configuration could lead to incorrect safety indicator rendering |
| **Verification** | Test with corrupted DDH data; verify error reported |
| **Derived TSRs** | TSR-DD-001, TSR-DD-002 |

#### FSR-DD-002: DDH Version Verification

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-DD-002 |
| **Description** | The LSR shall verify that the DDH binary version matches the expected version and reject incompatible configurations. |
| **ASIL** | D |
| **Derived From** | SG1 |
| **FTTI** | N/A (startup only) |
| **Safe State** | Engine reports LSR_DB_DDHBIN_VERSION_MISMATCH |
| **Rationale** | Version mismatch could lead to incorrect interpretation of configuration data |
| **Verification** | Test with mismatched DDH versions; verify rejection |
| **Derived TSRs** | TSR-DD-003 |

#### FSR-DD-003: Bitmap Data Integrity

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-DD-003 |
| **Description** | The LSR shall verify bitmap data integrity before rendering safety-critical indicators. |
| **ASIL** | D |
| **Derived From** | SG1 |
| **FTTI** | 100 ms |
| **Safe State** | Display known-safe pattern; report error |
| **Rationale** | Corrupted bitmap data results in incorrect visual presentation |
| **Verification** | Fault injection of corrupted bitmap; verify detection |
| **Derived TSRs** | TSR-DD-004, TSR-DD-005 |

#### FSR-DD-004: Bitmap ID Validation

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-DD-004 |
| **Description** | The LSR shall validate bitmap IDs against the configured range and reject invalid IDs. |
| **ASIL** | D |
| **Derived From** | SG1 |
| **FTTI** | 100 ms |
| **Safe State** | Omit rendering of invalid bitmap; report error |
| **Rationale** | Invalid bitmap ID could result in wrong indicator or crash |
| **Verification** | Test with out-of-range bitmap IDs; verify rejection |
| **Derived TSRs** | TSR-DD-006 |

#### FSR-DD-005: Render Output Correctness

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-DD-005 |
| **Description** | The LSR shall render safety indicators at the correct screen position, size, and with correct pixel content as specified in the DDH configuration. |
| **ASIL** | D |
| **Derived From** | SG1 |
| **FTTI** | 100 ms |
| **Safe State** | Verified by FSR-VE-001 |
| **Rationale** | Misplaced or malformed indicators may not be recognized |
| **Verification** | Visual verification against reference; automated pixel comparison |
| **Derived TSRs** | TSR-DD-007, TSR-DD-008 |

---

### 3.2 Availability Requirements (FSR-AV)

#### FSR-AV-001: Render Cycle Completion

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-AV-001 |
| **Description** | The LSR shall complete each render cycle within the configured frame budget and report completion status. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | Configurable (default 100 ms) |
| **Safe State** | Report render failure; system enters degraded mode |
| **Rationale** | Incomplete rendering results in missing safety indicators |
| **Verification** | Measure render cycle duration; verify completion reporting |
| **Derived TSRs** | TSR-AV-001, TSR-AV-002 |

#### FSR-AV-002: Render Failure Detection

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-AV-002 |
| **Description** | The LSR shall detect and report rendering failures via the Engine::getError() interface within the FTTI. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | 100 ms |
| **Safe State** | Error code returned; integrator handles safe state |
| **Rationale** | Silent render failures result in undetected missing indicators |
| **Verification** | Inject render failures; verify error detection and reporting |
| **Derived TSRs** | TSR-AV-003, TSR-AV-004 |

#### FSR-AV-003: Safe State Entry

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-AV-003 |
| **Description** | Upon detection of an unrecoverable error, the LSR shall transition to a safe state by ceasing normal rendering and reporting the error. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | 100 ms |
| **Safe State** | No rendering; error code available |
| **Rationale** | Continued operation after critical failure may produce incorrect output |
| **Verification** | Inject critical errors; verify safe state entry |
| **Derived TSRs** | TSR-AV-005 |

#### FSR-AV-004: Widget Tree Integrity

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-AV-004 |
| **Description** | The LSR shall maintain the integrity of the widget tree structure and detect corruption that would prevent correct rendering. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | 100 ms |
| **Safe State** | Report corruption; cease rendering |
| **Rationale** | Corrupted widget tree leads to missing or incorrect indicators |
| **Verification** | Fault injection of widget tree corruption; verify detection |
| **Derived TSRs** | TSR-AV-006 |

---

### 3.3 Timing Requirements (FSR-TI)

#### FSR-TI-001: Maximum Render Latency

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-TI-001 |
| **Description** | The LSR shall complete the render operation within the configurable maximum latency budget. |
| **ASIL** | C |
| **Derived From** | SG3 |
| **FTTI** | Application-specific (default 100 ms) |
| **Safe State** | Report timing violation |
| **Rationale** | Late rendering delays critical safety information |
| **Verification** | Measure render latency under various loads; verify bounded timing |
| **Derived TSRs** | TSR-TI-001 |

#### FSR-TI-002: Timing Violation Reporting

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-TI-002 |
| **Description** | The LSR shall detect and report timing budget violations to the integration layer. |
| **ASIL** | C |
| **Derived From** | SG3 |
| **FTTI** | 100 ms |
| **Safe State** | Error reported; integrator handles response |
| **Rationale** | Timing violations must be detected for system-level handling |
| **Verification** | Induce timing violations; verify detection and reporting |
| **Derived TSRs** | TSR-TI-002 |

#### FSR-TI-003: Display Update Rate

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-TI-003 |
| **Description** | The LSR shall support a minimum display update rate of 10 Hz for safety-critical content. |
| **ASIL** | C |
| **Derived From** | SG3 |
| **FTTI** | 100 ms |
| **Safe State** | N/A (design requirement) |
| **Rationale** | Minimum update rate ensures timely indicator changes |
| **Verification** | Measure actual update rate; verify ≥10 Hz |
| **Derived TSRs** | TSR-TI-003 |

---

### 3.4 Verification Requirements (FSR-VE)

#### FSR-VE-001: Video Output Verification

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-VE-001 |
| **Description** | The LSR shall perform pixel-level video output verification comparing rendered output against reference bitmaps for safety-critical content. |
| **ASIL** | C |
| **Derived From** | SG4 |
| **FTTI** | 100 ms |
| **Safe State** | Report verification failure; increment error counter |
| **Rationale** | Detects display corruption not caught by other mechanisms |
| **Verification** | Inject pixel corruption; verify detection |
| **Derived TSRs** | TSR-VE-001, TSR-VE-002 |

#### FSR-VE-002: Diagnostic Coverage

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-VE-002 |
| **Description** | The video output verification shall achieve a diagnostic coverage of at least 99% for single-pixel corruption in safety-critical areas. |
| **ASIL** | C |
| **Derived From** | SG4 |
| **FTTI** | 100 ms |
| **Safe State** | N/A (coverage requirement) |
| **Rationale** | High diagnostic coverage ensures effective detection |
| **Verification** | Fault injection testing with statistical analysis |
| **Derived TSRs** | TSR-VE-003 |

#### FSR-VE-003: Verification Error Reporting

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-VE-003 |
| **Description** | The LSR shall report verification failures via the ReferenceBitmapField error counter and Engine error interface. |
| **ASIL** | C |
| **Derived From** | SG4 |
| **FTTI** | 100 ms |
| **Safe State** | Error reported; counter incremented |
| **Rationale** | Verification results must be accessible to integration layer |
| **Verification** | Verify error reporting path; test error counter |
| **Derived TSRs** | TSR-VE-004 |

#### FSR-VE-004: Verification Enablement

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-VE-004 |
| **Description** | The LSR shall perform verification only when the ReferenceBitmapField visible flag is enabled. |
| **ASIL** | C |
| **Derived From** | SG4 |
| **FTTI** | N/A |
| **Safe State** | N/A (control requirement) |
| **Rationale** | Provides control over verification activation |
| **Verification** | Test verification with visible flag true/false |
| **Derived TSRs** | TSR-VE-005 |

---

### 3.5 Memory Safety Requirements (FSR-MS)

#### FSR-MS-001: Pool Integrity Checking

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-MS-001 |
| **Description** | The LSR shall verify memory pool integrity before each allocation and deallocation operation using marker-based detection. |
| **ASIL** | D |
| **Derived From** | SG1, SG2 |
| **FTTI** | Immediate (per operation) |
| **Safe State** | Return LSR_POOL_IS_CORRUPTED; deny operation |
| **Rationale** | Memory corruption can lead to any failure mode |
| **Verification** | Inject marker corruption; verify detection |
| **Derived TSRs** | TSR-MS-001, TSR-MS-002 |

#### FSR-MS-002: Double Deallocation Detection

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-MS-002 |
| **Description** | The LSR shall detect and prevent double deallocation of memory pool objects. |
| **ASIL** | D |
| **Derived From** | SG1, SG2 |
| **FTTI** | Immediate (per operation) |
| **Safe State** | Return LSR_POOL_DOUBLE_DELETE; deny operation |
| **Rationale** | Double-free corrupts memory management structures |
| **Verification** | Attempt double deallocation; verify detection |
| **Derived TSRs** | TSR-MS-003 |

#### FSR-MS-003: Pool Exhaustion Handling

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-MS-003 |
| **Description** | The LSR shall detect pool exhaustion and return an appropriate error without causing undefined behavior. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | Immediate (per operation) |
| **Safe State** | Return LSR_POOL_IS_FULL; deny allocation |
| **Rationale** | Pool exhaustion must be handled gracefully |
| **Verification** | Exhaust pool; verify error return and no crash |
| **Derived TSRs** | TSR-MS-004 |

#### FSR-MS-004: Invalid Pointer Detection

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-MS-004 |
| **Description** | The LSR shall detect and reject deallocation requests for pointers not allocated from the pool. |
| **ASIL** | D |
| **Derived From** | SG1, SG2 |
| **FTTI** | Immediate (per operation) |
| **Safe State** | Return LSR_POOL_INVALID_OBJECT; deny operation |
| **Rationale** | Invalid pointer operations corrupt memory |
| **Verification** | Pass invalid pointers; verify rejection |
| **Derived TSRs** | TSR-MS-005 |

#### FSR-MS-005: No Dynamic Allocation

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-MS-005 |
| **Description** | The LSR shall not use dynamic memory allocation (malloc/new) at runtime; all objects shall be allocated from pre-sized pools. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | N/A (design constraint) |
| **Safe State** | N/A |
| **Rationale** | Dynamic allocation introduces fragmentation and timing uncertainty |
| **Verification** | Static analysis; runtime monitoring of heap |
| **Derived TSRs** | TSR-MS-006 |

---

### 3.6 Error Handling Requirements (FSR-ER)

#### FSR-ER-001: Hierarchical Error Collection

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-ER-001 |
| **Description** | The LSR shall collect errors from all components hierarchically and make the highest-severity error available via Engine::getError(). |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | 100 ms |
| **Safe State** | Error available for retrieval |
| **Rationale** | Comprehensive error visibility enables proper system response |
| **Verification** | Inject errors at various levels; verify propagation |
| **Derived TSRs** | TSR-ER-001, TSR-ER-002 |

#### FSR-ER-002: Error Code Classification

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-ER-002 |
| **Description** | The LSR shall classify errors by severity and domain using a defined error code scheme that allows identification of error source. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | N/A (design requirement) |
| **Safe State** | N/A |
| **Rationale** | Error classification enables appropriate response |
| **Verification** | Review error codes; verify domain identification |
| **Derived TSRs** | TSR-ER-003 |

#### FSR-ER-003: Assertion Failure Handling

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-ER-003 |
| **Description** | The LSR shall invoke pilAssert() upon detection of programming errors (assertion failures) to allow platform-specific error handling. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | Immediate |
| **Safe State** | Platform-defined response |
| **Rationale** | Assertions detect unexpected conditions requiring attention |
| **Verification** | Trigger assertion failures; verify pilAssert() invocation |
| **Derived TSRs** | TSR-ER-004 |

---

### 3.7 Initialization Requirements (FSR-IN)

#### FSR-IN-001: Engine Initialization

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-IN-001 |
| **Description** | The LSR Engine shall perform complete initialization including database loading, display setup, and widget tree construction before accepting render requests. |
| **ASIL** | D |
| **Derived From** | SG1, SG2 |
| **FTTI** | N/A (startup) |
| **Safe State** | Initialization error reported |
| **Rationale** | Incomplete initialization leads to undefined behavior |
| **Verification** | Verify initialization sequence; test with incomplete init |
| **Derived TSRs** | TSR-IN-001, TSR-IN-002 |

#### FSR-IN-002: Initialization Error Reporting

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-IN-002 |
| **Description** | The LSR shall report initialization failures via the error interface and prevent rendering until successful initialization. |
| **ASIL** | D |
| **Derived From** | SG2 |
| **FTTI** | N/A (startup) |
| **Safe State** | Error reported; render blocked |
| **Rationale** | Post-failure rendering produces undefined results |
| **Verification** | Inject init failures; verify render blocking |
| **Derived TSRs** | TSR-IN-003 |

---

### 3.8 False Indication Requirements (FSR-FI)

#### FSR-FI-001: Data Validity Checking

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-FI-001 |
| **Description** | The LSR shall validate input data status before rendering safety indicators; invalid or unavailable data shall not result in indicator display. |
| **ASIL** | A |
| **Derived From** | SG5 |
| **FTTI** | 500 ms |
| **Safe State** | Omit indicator; report data status |
| **Rationale** | Displaying indicators without valid data is misleading |
| **Verification** | Test with invalid data status; verify no display |
| **Derived TSRs** | TSR-FI-001, TSR-FI-002 |

#### FSR-FI-002: Unavailable Data Handling

| Attribute | Value |
|-----------|-------|
| **ID** | FSR-FI-002 |
| **Description** | When data is marked as NOT_AVAILABLE, the LSR shall not render the associated safety indicator. |
| **ASIL** | A |
| **Derived From** | SG5 |
| **FTTI** | 500 ms |
| **Safe State** | Indicator not displayed |
| **Rationale** | Prevents display of indicators based on unknown state |
| **Verification** | Set data to NOT_AVAILABLE; verify no rendering |
| **Derived TSRs** | TSR-FI-003 |

---

## 4. Requirements Summary

### 4.1 Requirements by Category

| Category | Count | ASIL D | ASIL C | ASIL A |
|----------|-------|--------|--------|--------|
| Data/Display (DD) | 5 | 5 | 0 | 0 |
| Availability (AV) | 4 | 4 | 0 | 0 |
| Timing (TI) | 3 | 0 | 3 | 0 |
| Verification (VE) | 4 | 0 | 4 | 0 |
| Memory Safety (MS) | 5 | 5 | 0 | 0 |
| Error Handling (ER) | 3 | 3 | 0 | 0 |
| Initialization (IN) | 2 | 2 | 0 | 0 |
| False Indication (FI) | 2 | 0 | 0 | 2 |
| **Total** | **28** | **19** | **7** | **2** |

### 4.2 Requirements by Safety Goal

| Safety Goal | Related FSRs |
|-------------|--------------|
| SG1 (Correct Display) | FSR-DD-001 to FSR-DD-005, FSR-MS-001, FSR-MS-002, FSR-MS-004, FSR-IN-001 |
| SG2 (Availability) | FSR-AV-001 to FSR-AV-004, FSR-MS-001 to FSR-MS-005, FSR-ER-001 to FSR-ER-003, FSR-IN-001, FSR-IN-002 |
| SG3 (Timeliness) | FSR-TI-001 to FSR-TI-003 |
| SG4 (Corruption Detection) | FSR-VE-001 to FSR-VE-004 |
| SG5 (No False Indication) | FSR-FI-001, FSR-FI-002 |

---

## 5. Traceability Matrix

### 5.1 Safety Goal to FSR Traceability

| SG | FSR-DD | FSR-AV | FSR-TI | FSR-VE | FSR-MS | FSR-ER | FSR-IN | FSR-FI |
|----|--------|--------|--------|--------|--------|--------|--------|--------|
| SG1 | 001-005 | - | - | - | 001,002,004 | - | 001 | - |
| SG2 | - | 001-004 | - | - | 001-005 | 001-003 | 001,002 | - |
| SG3 | - | - | 001-003 | - | - | - | - | - |
| SG4 | - | - | - | 001-004 | - | - | - | - |
| SG5 | - | - | - | - | - | - | - | 001,002 |

### 5.2 FSR to TSR Mapping

See LSR-TSR-001 for complete FSR to TSR traceability.

---

## 6. SEooC Interface Requirements

### 6.1 GIL Interface Requirements

| Req ID | Requirement | ASIL |
|--------|-------------|------|
| FSR-IF-GIL-001 | GIL implementation shall meet ASIL D requirements for rendering functions | D |
| FSR-IF-GIL-002 | GIL implementation shall meet ASIL C requirements for gilVerify() function | C |
| FSR-IF-GIL-003 | GIL shall report errors via GIL_INVALID_* error codes | D |

### 6.2 PIL Interface Requirements

| Req ID | Requirement | ASIL |
|--------|-------------|------|
| FSR-IF-PIL-001 | PIL implementation shall meet ASIL C requirements for pilGetMonotonicTime() | C |
| FSR-IF-PIL-002 | PIL implementation shall meet ASIL D requirements for pilAssert() | D |
| FSR-IF-PIL-003 | pilGetMonotonicTime() shall provide monotonic time with resolution ≤1 ms | C |

### 6.3 IHMI Interface Requirements

| Req ID | Requirement | ASIL |
|--------|-------------|------|
| FSR-IF-IHMI-001 | IHMI implementation shall provide valid Frame data for rendering | D |
| FSR-IF-IHMI-002 | IHMI shall indicate data validity status for safety-critical content | A |

---

## 7. Assumptions and Dependencies

### 7.1 SEooC Assumptions

| ID | Assumption | Verification at Integration |
|----|------------|-----------------------------|
| AS-FSR-001 | GIL correctly renders pixel data to hardware | Hardware-in-loop testing |
| AS-FSR-002 | PIL provides accurate monotonic time | Platform qualification |
| AS-FSR-003 | DDH data is generated by qualified tool | Tool qualification |
| AS-FSR-004 | Memory hardware is fault-free | Hardware qualification |
| AS-FSR-005 | IHMI provides correct frame configuration | Integration testing |

### 7.2 External Dependencies

| Dependency | Impact | Mitigation |
|------------|--------|------------|
| GIL implementation quality | Rendering correctness | Qualification requirement |
| PIL timing accuracy | Timing compliance | Platform testing |
| Hardware display | Visual output | Hardware qualification |

---

## Appendix A: Requirement Attributes Summary

| FSR ID | Description | ASIL | FTTI | Safe State |
|--------|-------------|------|------|------------|
| FSR-DD-001 | Configuration validation | D | N/A | Error |
| FSR-DD-002 | Version verification | D | N/A | Error |
| FSR-DD-003 | Bitmap integrity | D | 100ms | Safe pattern |
| FSR-DD-004 | Bitmap ID validation | D | 100ms | Omit + Error |
| FSR-DD-005 | Render correctness | D | 100ms | Verification |
| FSR-AV-001 | Render completion | D | 100ms | Degraded |
| FSR-AV-002 | Failure detection | D | 100ms | Error |
| FSR-AV-003 | Safe state entry | D | 100ms | No render |
| FSR-AV-004 | Widget integrity | D | 100ms | Error |
| FSR-TI-001 | Max latency | C | Config | Report |
| FSR-TI-002 | Timing violation | C | 100ms | Report |
| FSR-TI-003 | Update rate | C | 100ms | N/A |
| FSR-VE-001 | Video verification | C | 100ms | Report |
| FSR-VE-002 | Diagnostic coverage | C | 100ms | N/A |
| FSR-VE-003 | Error reporting | C | 100ms | Report |
| FSR-VE-004 | Verification control | C | N/A | N/A |
| FSR-MS-001 | Pool integrity | D | Immed | Error |
| FSR-MS-002 | Double delete | D | Immed | Error |
| FSR-MS-003 | Exhaustion | D | Immed | Error |
| FSR-MS-004 | Invalid pointer | D | Immed | Error |
| FSR-MS-005 | No dynamic alloc | D | N/A | N/A |
| FSR-ER-001 | Error collection | D | 100ms | Available |
| FSR-ER-002 | Error classification | D | N/A | N/A |
| FSR-ER-003 | Assertion handling | D | Immed | Platform |
| FSR-IN-001 | Engine init | D | N/A | Error |
| FSR-IN-002 | Init error report | D | N/A | Blocked |
| FSR-FI-001 | Data validity | A | 500ms | Omit |
| FSR-FI-002 | Unavailable data | A | 500ms | Omit |

---

**End of Document**
