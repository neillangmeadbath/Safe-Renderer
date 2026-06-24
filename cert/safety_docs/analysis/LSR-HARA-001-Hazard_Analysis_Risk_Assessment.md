# LSR-HARA-001: Hazard Analysis and Risk Assessment

| Document ID | LSR-HARA-001 |
|-------------|--------------|
| Version | 1.0 |
| Date | 2026-05-12 |
| Status | Draft |
| Classification | Safety-Critical |
| Standard | ISO 26262:2018 Part 3 |
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
| ISO 26262:2018 | Road vehicles - Functional safety |
| LSR-SAD-001 | Software Architecture Description |
| LSR-FSR-001 | Functional Safety Requirements |

---

## 1. Introduction

### 1.1 Purpose

This document presents the Hazard Analysis and Risk Assessment (HARA) for the Luxoft Safe Renderer (LSR) software component. The HARA is performed in accordance with ISO 26262:2018 Part 3 to:

1. Identify and classify hazardous events
2. Assess associated risks using Severity, Exposure, and Controllability
3. Determine Automotive Safety Integrity Levels (ASIL)
4. Derive safety goals to prevent or mitigate hazardous events

### 1.2 Scope

This HARA covers the Luxoft Safe Renderer as a Safety Element out of Context (SEooC) intended for integration into automotive HMI systems. The scope includes:

**In Scope:**
- Core rendering engine (`engine/lsr`)
- Database management (`engine/database`)
- Display management (`engine/display`)
- Frame handling (`engine/framehandler`)
- Common utilities (`engine/common`)
- Graphics Interface Layer (`gil`)
- Platform Interface Layer (`pil`)

**Out of Scope:**
- Simulation modules (`simu/`)
- Third-party test frameworks (`3rdparty/`)
- Customer HMI application code
- Hardware platform specifics

### 1.3 SEooC Assumptions

As a Safety Element out of Context, the following assumptions apply:

| ID | Assumption | Rationale |
|----|------------|-----------|
| A1 | LSR is integrated into a vehicle display system (instrument cluster, head unit) | Primary deployment context |
| A2 | LSR renders safety-critical visual indicators (warning lamps, telltales) | Core safety function |
| A3 | Driver relies on displayed information for safe vehicle operation | Justifies safety-critical classification |
| A4 | Integration environment provides compliant hardware and platform services | SEooC boundary assumption |
| A5 | GIL and PIL implementations are provided by integrator with appropriate ASIL | Interface compliance |

---

## 2. Item Definition

### 2.1 Item Description

The Luxoft Safe Renderer (LSR) is a safety-critical HMI rendering engine designed for automotive applications. It provides:

1. **Rendering of Safety-Critical Graphics**: Display warning indicators, telltales, and safety-related visual information
2. **Video Output Verification**: Compare rendered output against reference bitmaps to detect corruption
3. **Fallback Rendering**: Take over display duties if the main HMI system fails
4. **Deterministic Operation**: Pre-allocated memory, bounded execution times

### 2.2 Item Boundary

```
┌─────────────────────────────────────────────────────────────────┐
│                     VEHICLE SYSTEM                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    HMI SYSTEM                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │           LUXOFT SAFE RENDERER (LSR)                │  │  │
│  │  │  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐  │  │  │
│  │  │  │ Engine  │ │Database │ │ Display  │ │FrameHndlr│  │  │  │
│  │  │  └────┬────┘ └────┬────┘ └────┬─────┘ └────┬─────┘  │  │  │
│  │  │       │           │           │            │         │  │  │
│  │  │  ┌────┴───────────┴───────────┴────────────┴─────┐  │  │  │
│  │  │  │                Common Utilities                │  │  │  │
│  │  │  └────────────────────────────────────────────────┘  │  │  │
│  │  └──────────────────────┬───────────────────────────────┘  │  │
│  │                         │                                   │  │
│  │  ┌──────────────────────┼───────────────────────────────┐  │  │
│  │  │      INTEGRATION BOUNDARY (GIL/PIL Interfaces)       │  │  │
│  │  └──────────────────────┼───────────────────────────────┘  │  │
│  │                         │                                   │  │
│  │  ┌──────────────────────┴───────────────────────────────┐  │  │
│  │  │     Platform Services (Graphics HW, Timers, etc.)    │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    DISPLAY HARDWARE                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                         DRIVER                             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Item Functions

| Function ID | Function Name | Description |
|-------------|---------------|-------------|
| F1 | Render | Render graphical content to display buffer |
| F2 | Verify | Compare rendered output against reference bitmap |
| F3 | HandleEvents | Process window and display events |
| F4 | ErrorReport | Collect and report error status |
| F5 | Initialize | Initialize rendering engine and load configuration |

### 2.4 Item Interfaces

| Interface | Direction | Description | Safety Relevance |
|-----------|-----------|-------------|------------------|
| IHMI | Input | Customer HMI data provider | Provides frame content |
| DDH | Input | Display Definition Hardware configuration | Static configuration |
| GIL | Output | Graphics Interface Layer | Renders to hardware |
| PIL | Input | Platform Interface Layer | System services |
| Error | Output | Error status reporting | Fault detection |

---

## 3. Operational Situations

### 3.1 Operational Modes

| Mode ID | Mode Name | Description |
|---------|-----------|-------------|
| OP1 | Normal Driving | Vehicle in motion, driver monitoring displays |
| OP2 | Standstill | Vehicle stationary, engine running |
| OP3 | Startup | System initialization, displays coming online |
| OP4 | Shutdown | System shutdown, displays being deactivated |
| OP5 | Emergency | Emergency situation requiring immediate driver attention |
| OP6 | Degraded | Main HMI failed, LSR operating as fallback |
| OP7 | Parking | Vehicle parked, reduced driver attention |

### 3.2 Environmental Conditions

| Condition ID | Condition | Impact on Operation |
|--------------|-----------|---------------------|
| ENV1 | Day/bright ambient light | Display brightness requirements |
| ENV2 | Night/dark ambient | Low brightness, high contrast requirements |
| ENV3 | Extreme temperature | Hardware performance variation |
| ENV4 | Vibration | Display stability requirements |
| ENV5 | EMC interference | Potential display corruption |

### 3.3 Use Cases

| UC ID | Use Case | Operational Mode | Description |
|-------|----------|------------------|-------------|
| UC1 | Warning Lamp Display | OP1, OP2, OP5 | Display critical warning indicators |
| UC2 | Telltale Rendering | OP1, OP2 | Display vehicle status telltales |
| UC3 | Fallback Mode | OP6 | LSR takes over from failed main HMI |
| UC4 | System Boot | OP3 | Initial display of safety indicators |
| UC5 | Continuous Verification | OP1, OP2 | Ongoing video output verification |

---

## 4. Hazard Identification

### 4.1 Malfunctioning Behavior Analysis

Analysis of potential malfunctioning behaviors for each item function:

| Function | Malfunction Type | Malfunctioning Behavior |
|----------|------------------|-------------------------|
| F1 Render | Commission | Incorrect graphic rendered (wrong indicator) |
| F1 Render | Omission | Graphic not rendered (missing indicator) |
| F1 Render | Timing | Graphic rendered late (delayed warning) |
| F1 Render | Value | Graphic corrupted (unreadable indicator) |
| F2 Verify | Commission | False positive (reports error when none exists) |
| F2 Verify | Omission | False negative (fails to detect corruption) |
| F3 HandleEvents | Omission | Display freeze (no updates) |
| F4 ErrorReport | Omission | Error not reported (silent failure) |
| F5 Initialize | Commission | Incorrect initialization (wrong config) |
| F5 Initialize | Omission | Initialization failure (no display) |

### 4.2 Hazard Catalog

| Hazard ID | Hazard Description | Causal Malfunctions |
|-----------|--------------------|--------------------|
| H1 | Incorrect safety warning displayed | F1-Commission, F5-Commission |
| H2 | Safety warning not displayed | F1-Omission, F3-Omission, F5-Omission |
| H3 | Safety warning displayed late | F1-Timing |
| H4 | Safety warning corrupted/unreadable | F1-Value |
| H5 | Display corruption undetected | F2-Omission, F4-Omission |
| H6 | System indicates false warning | F1-Commission, F2-Commission |
| H7 | Display freeze during critical situation | F3-Omission |

---

## 5. Hazardous Event Classification

### 5.1 Severity Classification (S)

Per ISO 26262-3, Table 1:

| Class | Description | Criteria |
|-------|-------------|----------|
| S0 | No injuries | No injuries to vehicle occupants or other road users |
| S1 | Light and moderate injuries | Injuries that are not life-threatening and from which recovery is expected |
| S2 | Severe and life-threatening injuries (survival probable) | Life-threatening injuries where survival is probable |
| S3 | Life-threatening injuries (survival uncertain), fatal injuries | Survival is uncertain or not expected |

### 5.2 Exposure Classification (E)

Per ISO 26262-3, Table 2:

| Class | Description | Probability |
|-------|-------------|-------------|
| E0 | Incredible | Probability negligible |
| E1 | Very low probability | < 1% of operating time |
| E2 | Low probability | 1% - 10% of operating time |
| E3 | Medium probability | 10% - 90% of operating time |
| E4 | High probability | > 90% of operating time |

### 5.3 Controllability Classification (C)

Per ISO 26262-3, Table 3:

| Class | Description | Criteria |
|-------|-------------|----------|
| C0 | Controllable in general | > 99% of drivers can avoid harm |
| C1 | Simply controllable | 99% of drivers can avoid harm |
| C2 | Normally controllable | 90% - 99% of drivers can avoid harm |
| C3 | Difficult to control or uncontrollable | < 90% of drivers can avoid harm |

### 5.4 ASIL Determination

Per ISO 26262-3, Table 4:

| Severity | Exposure | C1 | C2 | C3 |
|----------|----------|----|----|----|
| S1 | E1 | QM | QM | QM |
| S1 | E2 | QM | QM | QM |
| S1 | E3 | QM | QM | A |
| S1 | E4 | QM | A | B |
| S2 | E1 | QM | QM | QM |
| S2 | E2 | QM | QM | A |
| S2 | E3 | QM | A | B |
| S2 | E4 | A | B | C |
| S3 | E1 | QM | QM | A |
| S3 | E2 | QM | A | B |
| S3 | E3 | A | B | C |
| S3 | E4 | B | C | D |

---

## 6. Hazardous Event Assessment

### 6.1 HE1: Incorrect Safety Warning Displayed

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H1 | |
| **Description** | Incorrect safety warning displayed (e.g., wrong telltale, misleading indicator) | |
| **Operational Situation** | OP1 Normal Driving, OP5 Emergency | |
| **Scenario** | Driver sees incorrect brake system warning leading to improper braking technique | |
| **Severity** | **S3** | Incorrect safety information could lead to fatal accident |
| **Exposure** | **E4** | Safety warnings displayed continuously during vehicle operation |
| **Controllability** | **C3** | Driver cannot detect incorrect information; may rely on false data |
| **ASIL** | **D** | S3 + E4 + C3 = ASIL D |

### 6.2 HE2: Safety Warning Not Displayed (Missing)

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H2 | |
| **Description** | Critical safety warning fails to appear (e.g., ABS warning, engine overheat) | |
| **Operational Situation** | OP1 Normal Driving, OP5 Emergency | |
| **Scenario** | Brake system failure occurs but no warning displayed; driver unaware of degraded braking | |
| **Severity** | **S3** | Missing critical warning could lead to fatal accident |
| **Exposure** | **E4** | Safety indicators monitored continuously |
| **Controllability** | **C3** | Driver cannot know about condition without warning |
| **ASIL** | **D** | S3 + E4 + C3 = ASIL D |

### 6.3 HE3: Safety Warning Displayed Late

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H3 | |
| **Description** | Safety warning appears too late to allow driver reaction | |
| **Operational Situation** | OP1 Normal Driving, OP5 Emergency | |
| **Scenario** | Collision warning delayed by 500ms; insufficient time for avoidance |
| **Severity** | **S3** | Delayed warning could result in unavoidable collision |
| **Exposure** | **E3** | Time-critical warnings occur occasionally |
| **Controllability** | **C3** | Delayed warning removes driver's ability to react |
| **ASIL** | **C** | S3 + E3 + C3 = ASIL C |

### 6.4 HE4: Safety Warning Corrupted/Unreadable

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H4 | |
| **Description** | Safety warning rendered but corrupted, garbled, or unreadable | |
| **Operational Situation** | OP1 Normal Driving | |
| **Scenario** | Graphical corruption makes warning symbol unrecognizable |
| **Severity** | **S3** | Unreadable warning equivalent to missing warning |
| **Exposure** | **E3** | Display corruption possible during operation |
| **Controllability** | **C2** | Driver may notice corruption and seek other indicators |
| **ASIL** | **B** | S3 + E3 + C2 = ASIL B |

### 6.5 HE5: Display Corruption Undetected

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H5 | |
| **Description** | Video output verification fails to detect corruption | |
| **Operational Situation** | OP1 Normal Driving | |
| **Scenario** | Pixel verification mechanism fails; corrupted display goes unnoticed |
| **Severity** | **S3** | Leads to scenarios HE1-HE4 being undetected |
| **Exposure** | **E3** | Verification runs continuously but failures rare |
| **Controllability** | **C3** | No mechanism to detect verification failure |
| **ASIL** | **C** | S3 + E3 + C3 = ASIL C |

### 6.6 HE6: False Warning Displayed

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H6 | |
| **Description** | Warning displayed when no actual condition exists | |
| **Operational Situation** | OP1 Normal Driving | |
| **Scenario** | False brake warning causes driver to brake unnecessarily, causing rear-end collision |
| **Severity** | **S2** | Sudden unexpected braking can cause accidents |
| **Exposure** | **E3** | False positives occur occasionally |
| **Controllability** | **C2** | Driver may doubt false warning based on other factors |
| **ASIL** | **A** | S2 + E3 + C2 = ASIL A |

### 6.7 HE7: Display Freeze During Critical Situation

| Attribute | Value | Justification |
|-----------|-------|---------------|
| **Hazard ID** | H7 | |
| **Description** | Display stops updating, showing stale information | |
| **Operational Situation** | OP1 Normal Driving, OP5 Emergency | |
| **Scenario** | Display freezes; new warning conditions not displayed |
| **Severity** | **S3** | Frozen display equivalent to missing new warnings |
| **Exposure** | **E3** | System freeze possible during operation |
| **Controllability** | **C3** | Driver cannot detect frozen state |
| **ASIL** | **C** | S3 + E3 + C3 = ASIL C |

---

## 7. Hazardous Event Summary

| HE ID | Hazard | Severity | Exposure | Controllability | ASIL |
|-------|--------|----------|----------|-----------------|------|
| HE1 | Incorrect safety warning displayed | S3 | E4 | C3 | **D** |
| HE2 | Safety warning not displayed | S3 | E4 | C3 | **D** |
| HE3 | Safety warning displayed late | S3 | E3 | C3 | **C** |
| HE4 | Safety warning corrupted | S3 | E3 | C2 | **B** |
| HE5 | Display corruption undetected | S3 | E3 | C3 | **C** |
| HE6 | False warning displayed | S2 | E3 | C2 | **A** |
| HE7 | Display freeze | S3 | E3 | C3 | **C** |

**Maximum ASIL: D** (from HE1 and HE2)

---

## 8. Safety Goals

Based on the hazardous event analysis, the following safety goals are derived:

### 8.1 SG1: Correct Display of Safety Indicators

| Attribute | Value |
|-----------|-------|
| **Safety Goal ID** | SG1 |
| **Description** | The LSR shall correctly display all safety-critical indicators as specified |
| **ASIL** | D |
| **Safe State** | Display known-safe pattern or blank display |
| **Fault Tolerant Time Interval (FTTI)** | 100 ms (one frame at 10 Hz update rate) |
| **Related Hazards** | HE1, HE4 |

### 8.2 SG2: Availability of Safety Indicators

| Attribute | Value |
|-----------|-------|
| **Safety Goal ID** | SG2 |
| **Description** | The LSR shall display all required safety indicators without omission |
| **ASIL** | D |
| **Safe State** | Display known-safe pattern indicating system fault |
| **Fault Tolerant Time Interval (FTTI)** | 100 ms |
| **Related Hazards** | HE2, HE7 |

### 8.3 SG3: Timeliness of Safety Indicators

| Attribute | Value |
|-----------|-------|
| **Safety Goal ID** | SG3 |
| **Description** | The LSR shall display safety indicators within the specified timing budget |
| **ASIL** | C |
| **Safe State** | N/A (timing violation detected and reported) |
| **Fault Tolerant Time Interval (FTTI)** | Application-specific (typically 100-500 ms) |
| **Related Hazards** | HE3 |

### 8.4 SG4: Detection of Display Corruption

| Attribute | Value |
|-----------|-------|
| **Safety Goal ID** | SG4 |
| **Description** | The LSR shall detect display output corruption with specified diagnostic coverage |
| **ASIL** | C |
| **Safe State** | Report verification failure to system |
| **Fault Tolerant Time Interval (FTTI)** | 100 ms |
| **Diagnostic Coverage** | > 99% for single-pixel corruption |
| **Related Hazards** | HE4, HE5 |

### 8.5 SG5: Avoidance of False Indications

| Attribute | Value |
|-----------|-------|
| **Safety Goal ID** | SG5 |
| **Description** | The LSR shall not display safety indicators without valid data |
| **ASIL** | A |
| **Safe State** | Omit display if data validity uncertain |
| **Fault Tolerant Time Interval (FTTI)** | 500 ms |
| **Related Hazards** | HE6 |

---

## 9. Safety Goal Summary and Traceability

### 9.1 Safety Goal to Hazard Traceability

| Safety Goal | Related Hazardous Events | ASIL |
|-------------|-------------------------|------|
| SG1 | HE1, HE4 | D |
| SG2 | HE2, HE7 | D |
| SG3 | HE3 | C |
| SG4 | HE4, HE5 | C |
| SG5 | HE6 | A |

### 9.2 Safety Goal to Function Traceability

| Safety Goal | Related Functions | Safety Mechanism Required |
|-------------|-------------------|---------------------------|
| SG1 | F1 Render, F5 Initialize | Data validation, configuration verification |
| SG2 | F1 Render, F3 HandleEvents | Redundant rendering path, watchdog |
| SG3 | F1 Render | Execution time monitoring |
| SG4 | F2 Verify | Video output comparison |
| SG5 | F1 Render, F4 ErrorReport | Input data validation |

---

## 10. Functional Safety Requirements (Preliminary)

Based on the safety goals, the following preliminary functional safety requirements are derived. Full elaboration is in LSR-FSR-001.

### 10.1 FSR from SG1 (Correct Display)

| FSR ID | Requirement | ASIL | Derived From |
|--------|-------------|------|--------------|
| FSR-DD-001 | LSR shall validate configuration data (DDH) integrity at startup | D | SG1 |
| FSR-DD-002 | LSR shall verify bitmap data integrity before rendering | D | SG1 |
| FSR-DD-003 | LSR shall compare rendered output against reference for safety indicators | D | SG1, SG4 |

### 10.2 FSR from SG2 (Availability)

| FSR ID | Requirement | ASIL | Derived From |
|--------|-------------|------|--------------|
| FSR-AV-001 | LSR shall complete render cycle within specified frame budget | D | SG2 |
| FSR-AV-002 | LSR shall detect and report rendering failures | D | SG2 |
| FSR-AV-003 | LSR shall enter safe state upon detection of unrecoverable error | D | SG2 |

### 10.3 FSR from SG3 (Timeliness)

| FSR ID | Requirement | ASIL | Derived From |
|--------|-------------|------|--------------|
| FSR-TI-001 | LSR shall complete render operation within configurable time budget | C | SG3 |
| FSR-TI-002 | LSR shall report timing violations to the integration layer | C | SG3 |

### 10.4 FSR from SG4 (Corruption Detection)

| FSR ID | Requirement | ASIL | Derived From |
|--------|-------------|------|--------------|
| FSR-VE-001 | LSR shall perform video output verification at configurable intervals | C | SG4 |
| FSR-VE-002 | LSR shall detect single-pixel corruption with >99% diagnostic coverage | C | SG4 |
| FSR-VE-003 | LSR shall report verification failures via error interface | C | SG4 |

### 10.5 FSR from SG5 (No False Indications)

| FSR ID | Requirement | ASIL | Derived From |
|--------|-------------|------|--------------|
| FSR-FI-001 | LSR shall validate input data status before rendering | A | SG5 |
| FSR-FI-002 | LSR shall not render safety indicator if data validity is NOT_AVAILABLE | A | SG5 |

---

## 11. Assumptions and Constraints

### 11.1 SEooC Assumptions to be Validated at Integration

| ID | Assumption | Validation Method |
|----|------------|-------------------|
| AVI-01 | Platform provides monotonic time with resolution ≤ 1ms | Integration test |
| AVI-02 | Graphics hardware correctly renders pixel data | GIL qualification |
| AVI-03 | Memory is not corrupted by external factors | System-level safety analysis |
| AVI-04 | Customer IHMI implementation provides correct frame data | Customer responsibility |
| AVI-05 | DDH configuration is generated by qualified tool | Tool qualification |

### 11.2 Constraints on Integration

| ID | Constraint | Rationale |
|----|------------|-----------|
| CI-01 | Integrator shall ensure GIL implementation meets ASIL D | Interface safety |
| CI-02 | Integrator shall ensure PIL implementation meets ASIL D | Interface safety |
| CI-03 | System shall provide hardware watchdog | Hung state detection |
| CI-04 | Display hardware shall support pixel readback | Verification requirement |

---

## 12. Conclusion

This HARA identifies 7 hazardous events for the Luxoft Safe Renderer, with 2 events (HE1, HE2) classified as ASIL D. Five safety goals are derived to address these hazards:

| Safety Goal | ASIL | Summary |
|-------------|------|---------|
| SG1 | D | Correct display of safety indicators |
| SG2 | D | Availability of safety indicators |
| SG3 | C | Timeliness of safety indicators |
| SG4 | C | Detection of display corruption |
| SG5 | A | Avoidance of false indications |

The LSR is therefore classified as a **maximum ASIL D** component, requiring the most rigorous development and verification processes per ISO 26262.

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| ASIL | Automotive Safety Integrity Level |
| DDH | Display Definition Hardware (configuration data) |
| FTTI | Fault Tolerant Time Interval |
| GIL | Graphics Interface Layer |
| HARA | Hazard Analysis and Risk Assessment |
| HMI | Human Machine Interface |
| LSR | Luxoft Safe Renderer |
| PIL | Platform Interface Layer |
| SEooC | Safety Element out of Context |
| Telltale | Illuminated indicator symbol on vehicle dashboard |

## Appendix B: Referenced Standards

| Standard | Title |
|----------|-------|
| ISO 26262:2018 Part 1 | Vocabulary |
| ISO 26262:2018 Part 2 | Management of functional safety |
| ISO 26262:2018 Part 3 | Concept phase |
| ISO 26262:2018 Part 4 | Product development at the system level |
| ISO 26262:2018 Part 6 | Product development at the software level |
| ISO 26262:2018 Part 10 | Guideline on ISO 26262 |

---

**End of Document**
