# LSR-CFA-001: Control Flow and Data Flow Analysis

| Document ID | LSR-CFA-001 |
|-------------|--------------|
| Version | 1.0 |
| Date | 2026-05-12 |
| Status | Draft |
| Classification | Safety-Critical |
| Standard | ISO 26262:2018 Part 6 |

---

## 1. Introduction

This document presents control flow and data flow analysis for safety-critical functions in the Luxoft Safe Renderer. The analysis identifies:
- Execution paths through critical functions
- Data dependencies and transformations
- Potential safety-relevant paths
- Unreachable code analysis

---

## 2. Control Flow Analysis

### 2.1 Engine::Engine() Constructor - Initialization Flow

**Source**: `engine/lsr/src/Engine.cpp:24-45`

```mermaid
flowchart TD
    START([Engine Constructor Start]) --> INIT_DB[Initialize m_db with DDH]
    INIT_DB --> INIT_DISPLAY[Initialize m_display]
    INIT_DISPLAY --> INIT_FH[Initialize m_frameHandler]
    INIT_FH --> GET_DB_ERR[m_error = m_db.getError]

    GET_DB_ERR --> CHECK_ERR{m_error == LSR_NO_ENGINE_ERROR?}

    CHECK_ERR -->|No| END_ERR([Constructor End - Error State])
    CHECK_ERR -->|Yes| LOAD_TEX[m_display.loadAllTextures]

    LOAD_TEX --> CHECK_TEX{loadAllTextures succeeded?}

    CHECK_TEX -->|No| SET_INCONSISTENT[m_error = LSR_DB_INCONSISTENT]
    SET_INCONSISTENT --> END_ERR

    CHECK_TEX -->|Yes| START_FH[m_frameHandler.start]

    START_FH --> CHECK_START{start succeeded?}

    CHECK_START -->|No| SET_FH_ERR[m_error = m_frameHandler.getError]
    SET_FH_ERR --> END_ERR

    CHECK_START -->|Yes| END_OK([Constructor End - Success])

    style START fill:#90EE90
    style END_OK fill:#90EE90
    style END_ERR fill:#FFB6C1
    style SET_INCONSISTENT fill:#FFB6C1
    style SET_FH_ERR fill:#FFB6C1
```

**Critical Paths**:
| Path ID | Condition | Result | ASIL Impact |
|---------|-----------|--------|-------------|
| P1 | DB error at startup | Error state retained | SG2 - Availability |
| P2 | Texture load failure | LSR_DB_INCONSISTENT | SG1 - Correct Display |
| P3 | FrameHandler start failure | Component error | SG2 - Availability |
| P4 | All checks pass | Successful init | Normal operation |

---

### 2.2 Engine::getError() - Error Aggregation Flow

**Source**: `engine/lsr/src/Engine.cpp:62-83`

```mermaid
flowchart TD
    START([getError Start]) --> CREATE_ERR[err = Error m_error]
    CREATE_ERR --> CHECK_M_ERR{err.isError?}

    CHECK_M_ERR -->|Yes| CLEAR_M_ERR[m_error = LSR_NO_ENGINE_ERROR]
    CLEAR_M_ERR --> RETURN_ERR([Return err])

    CHECK_M_ERR -->|No| GET_DB_ERR[err = Error m_db.getError]
    GET_DB_ERR --> CHECK_DB_ERR{err.isError?}

    CHECK_DB_ERR -->|Yes| RETURN_ERR

    CHECK_DB_ERR -->|No| GET_FH_ERR[err = Error m_frameHandler.getError]
    GET_FH_ERR --> CHECK_FH_ERR{err.isError?}

    CHECK_FH_ERR -->|Yes| RETURN_ERR

    CHECK_FH_ERR -->|No| GET_DSP_ERR[err = Error m_display.getError]
    GET_DSP_ERR --> RETURN_ERR

    style START fill:#90EE90
    style RETURN_ERR fill:#87CEEB
```

**Error Priority Order**:
1. Engine-level error (m_error) - highest priority
2. Database error (m_db.getError())
3. FrameHandler error (m_frameHandler.getError())
4. Display error (m_display.getError()) - lowest priority

---

### 2.3 ReferenceBitmapField::onVerify() - Verification Flow

**Source**: `engine/framehandler/src/ReferenceBitmapField.cpp:61-84`

```mermaid
flowchart TD
    START([onVerify Start]) --> CHECK_VIS[m_verified = !isVisible]

    CHECK_VIS --> IS_INVISIBLE{m_verified == true?<br/>i.e., NOT visible}

    IS_INVISIBLE -->|Yes - Invisible| CLEAR_ERRORS[clearVerificationErrors]
    CLEAR_ERRORS --> RETURN_TRUE([Return m_verified = true])

    IS_INVISIBLE -->|No - Visible| GET_BITMAP[bitmap = m_pDatabase->getBitmap]
    GET_BITMAP --> DO_VERIFY[m_verified = dst.verify bitmap, rect]

    DO_VERIFY --> CHECK_VERIFY{m_verified?}

    CHECK_VERIFY -->|Yes| RETURN_VERIFIED([Return m_verified = true])

    CHECK_VERIFY -->|No - Verification Failed| CHECK_OVERFLOW{m_verificationErrors < U32_MAX?}

    CHECK_OVERFLOW -->|Yes| INC_ERRORS[++m_verificationErrors]
    INC_ERRORS --> RETURN_FAILED([Return m_verified = false])

    CHECK_OVERFLOW -->|No - Counter Saturated| RETURN_FAILED

    style START fill:#90EE90
    style RETURN_TRUE fill:#90EE90
    style RETURN_VERIFIED fill:#90EE90
    style RETURN_FAILED fill:#FFB6C1
    style DO_VERIFY fill:#FFFF99
```

**Safety-Critical Paths**:
| Path | Condition | Outcome | Safety Relevance |
|------|-----------|---------|------------------|
| Invisible Path | Field not visible | Skip verification, clear errors | Intentional bypass |
| Success Path | Pixel match | Return true | Normal operation |
| Failure Path | Pixel mismatch | Increment counter, return false | **SG4 - Corruption Detection** |
| Overflow Path | Counter at max | No increment, return false | Counter saturation handling |

---

### 2.4 Canvas::verify() - Pixel Verification Flow

**Source**: `engine/display/src/Canvas.cpp:76-106`

```mermaid
flowchart TD
    START([verify Start]) --> INIT_VERIFIED[verified = false]
    INIT_VERIFIED --> LOAD_TEX[t = m_dsp.loadTexture bitmap]

    LOAD_TEX --> CHECK_TEX{t != NULL?}

    CHECK_TEX -->|No| SET_ERROR[m_error = LSR_ERROR_NO_TEXTURE]
    SET_ERROR --> RETURN_FALSE([Return verified = false])

    CHECK_TEX -->|Yes| GET_CTX[ctx = m_dsp.getContext]
    GET_CTX --> BIND_TEX[t->bind ctx]

    BIND_TEX --> CALC_COORDS[Calculate x1,y1,x2,y2 from rect<br/>Calculate u1,v1,u2,v2 from texture]

    CALC_COORDS --> CALL_GIL[res = gilVerify ctx, coords]

    CALL_GIL --> CHECK_RES{res == GIL_TRUE?}

    CHECK_RES -->|Yes| RETURN_TRUE([Return verified = true])
    CHECK_RES -->|No| RETURN_FALSE2([Return verified = false])

    style START fill:#90EE90
    style RETURN_TRUE fill:#90EE90
    style RETURN_FALSE fill:#FFB6C1
    style RETURN_FALSE2 fill:#FFB6C1
    style SET_ERROR fill:#FFB6C1
    style CALL_GIL fill:#FFFF99
```

---

### 2.5 Pool::allocate() - Memory Allocation Flow

**Source**: `engine/common/api/Pool.h:191-216`

```mermaid
flowchart TD
    START([allocate Start]) --> INIT_NULL[pData = NULL]
    INIT_NULL --> CHECK_POOL[checkPool]

    CHECK_POOL --> IS_VALID{checkPool == true?}

    IS_VALID -->|No - Corruption Detected| SET_CORRUPTED[error = LSR_POOL_IS_CORRUPTED]
    SET_CORRUPTED --> RETURN_NULL([Return NULL])

    IS_VALID -->|Yes - Pool OK| CHECK_FREE{m_pFreeList != NULL?}

    CHECK_FREE -->|No - Pool Exhausted| SET_FULL[error = LSR_POOL_IS_FULL]
    SET_FULL --> RETURN_NULL

    CHECK_FREE -->|Yes - Space Available| GET_DATA[pData = m_pFreeList->body.data]
    GET_DATA --> SET_MARKER[m_pFreeList->marker = m_markerBusy]
    SET_MARKER --> ADVANCE_LIST[m_pFreeList = m_pFreeList->body.next]
    ADVANCE_LIST --> SET_SUCCESS[error = LSR_NO_ENGINE_ERROR]
    SET_SUCCESS --> RETURN_DATA([Return pData])

    style START fill:#90EE90
    style RETURN_DATA fill:#90EE90
    style RETURN_NULL fill:#FFB6C1
    style SET_CORRUPTED fill:#FF0000,color:#FFF
    style SET_FULL fill:#FFB6C1
    style SET_MARKER fill:#FFFF99
```

**Safety Mechanisms**:
| Check | Purpose | Error Code |
|-------|---------|------------|
| checkPool() | Detect memory corruption | LSR_POOL_IS_CORRUPTED |
| m_pFreeList != NULL | Detect exhaustion | LSR_POOL_IS_FULL |
| Marker update | Track allocation state | 0x55 pattern |

---

### 2.6 Pool::deallocate() - Memory Deallocation Flow

**Source**: `engine/common/api/Pool.h:218-256`

```mermaid
flowchart TD
    START([deallocate Start]) --> INIT_RES[res = LSR_NO_ENGINE_ERROR]
    INIT_RES --> CHECK_POOL[checkPool]

    CHECK_POOL --> IS_VALID{checkPool == true?}

    IS_VALID -->|No| SET_CORRUPTED[res = LSR_POOL_IS_CORRUPTED]
    SET_CORRUPTED --> RETURN([Return res])

    IS_VALID -->|Yes| CHECK_PTR{ptr != NULL AND isAllocated ptr?}

    CHECK_PTR -->|No| SET_INVALID[res = LSR_POOL_INVALID_OBJECT]
    SET_INVALID --> RETURN

    CHECK_PTR -->|Yes| CHECK_FREE{checkObjectIsFree ptr?}

    CHECK_FREE -->|Yes - Already Free| SET_DOUBLE[res = LSR_POOL_DOUBLE_DELETE]
    SET_DOUBLE --> RETURN

    CHECK_FREE -->|No - Properly Allocated| ZERO_MEM[memset pNode->body.data, 0, sizeof T]
    ZERO_MEM --> UPDATE_NEXT[pNode->body.next = m_pFreeList]
    UPDATE_NEXT --> SET_FREE_MARKER[pNode->marker = m_markerFree]
    SET_FREE_MARKER --> UPDATE_FREELIST[m_pFreeList = pNode]
    UPDATE_FREELIST --> RETURN_SUCCESS([Return LSR_NO_ENGINE_ERROR])

    style START fill:#90EE90
    style RETURN_SUCCESS fill:#90EE90
    style RETURN fill:#87CEEB
    style SET_CORRUPTED fill:#FF0000,color:#FFF
    style SET_INVALID fill:#FFB6C1
    style SET_DOUBLE fill:#FFB6C1
    style SET_FREE_MARKER fill:#FFFF99
```

**Multi-Level Validation**:
1. Pool integrity check (checkPool)
2. Pointer validity (ptr != NULL && isAllocated)
3. Double-free detection (checkObjectIsFree)
4. Memory zeroing before free (security measure)

---

## 3. Data Flow Analysis

### 3.1 Render Pipeline Data Flow

```mermaid
flowchart LR
    subgraph Input
        DDH[(DDH Config)]
        IHMI[IHMI Frame Data]
    end

    subgraph Engine
        DB[Database]
        FH[FrameHandler]
        DSP[DisplayManager]
    end

    subgraph Rendering
        WIN[Window]
        FRM[Frame]
        PNL[Panel]
        FLD[BitmapField]
    end

    subgraph Output
        CVS[Canvas]
        GIL[GIL Context]
        HW[Display Hardware]
    end

    DDH --> DB
    IHMI --> FH
    DB --> FH
    DB --> DSP
    FH --> WIN
    WIN --> FRM
    FRM --> PNL
    PNL --> FLD
    FLD --> CVS
    DSP --> CVS
    CVS --> GIL
    GIL --> HW
```

### 3.2 Verification Data Flow

```mermaid
flowchart LR
    subgraph Reference
        DDH[(DDH Config)]
        BMP[Reference Bitmap]
    end

    subgraph Verification
        RBF[ReferenceBitmapField]
        CVS[Canvas]
        TEX[Texture]
    end

    subgraph Comparison
        GIL_V[gilVerify]
        FB[Frame Buffer<br/>Actual Pixels]
    end

    subgraph Result
        VER{Verified?}
        ERR[Error Counter]
        OK[Success]
    end

    DDH --> BMP
    BMP --> RBF
    RBF --> CVS
    CVS --> TEX
    TEX --> GIL_V
    FB --> GIL_V
    GIL_V --> VER
    VER -->|No| ERR
    VER -->|Yes| OK
```

### 3.3 Error Propagation Data Flow

```mermaid
flowchart BT
    subgraph Sources
        POOL[Pool Errors]
        GIL_E[GIL Errors]
        DB_E[Database Errors]
    end

    subgraph Components
        DB[Database]
        DSP[DisplayManager]
        FH[FrameHandler]
    end

    subgraph Aggregation
        ENG[Engine]
        ERR_COL[Error Collector]
    end

    subgraph Output
        GET_ERR[Engine::getError]
        APP[Application]
    end

    POOL --> DB
    POOL --> DSP
    POOL --> FH
    GIL_E --> DSP
    DB_E --> DB

    DB --> ERR_COL
    DSP --> ERR_COL
    FH --> ERR_COL

    ERR_COL --> ENG
    ENG --> GET_ERR
    GET_ERR --> APP
```

### 3.4 Pool Memory Data Flow

```mermaid
flowchart TD
    subgraph Pool_Structure
        STORAGE[m_storage<br/>U8 array]
        FREELIST[m_pFreeList<br/>Node pointer]
        MARKERS[m_markerFree/Busy<br/>0xAA/0x55]
    end

    subgraph Allocate
        A_CHECK[checkPool]
        A_GET[Get from freelist]
        A_MARK[Set busy marker]
    end

    subgraph Deallocate
        D_CHECK[checkPool]
        D_VALID[Validate pointer]
        D_ZERO[Zero memory]
        D_MARK[Set free marker]
        D_RETURN[Return to freelist]
    end

    subgraph Validation
        IS_ALLOC[isAllocated]
        CHECK_BOUNDS[Bounds check]
        CHECK_MARKER[Marker check]
    end

    STORAGE --> A_CHECK
    FREELIST --> A_GET
    MARKERS --> A_MARK

    A_CHECK --> D_CHECK
    A_GET --> IS_ALLOC
    A_MARK --> CHECK_MARKER

    D_VALID --> CHECK_BOUNDS
    D_VALID --> CHECK_MARKER
    D_ZERO --> D_MARK
    D_MARK --> D_RETURN
    D_RETURN --> FREELIST
```

---

## 4. Critical Path Analysis

### 4.1 Safety-Critical Execution Paths

| Path ID | Function | Critical Decision | Safety Impact |
|---------|----------|-------------------|---------------|
| CP-001 | Engine::Engine | Texture load check | Display availability |
| CP-002 | Engine::getError | Error priority chain | Error visibility |
| CP-003 | ReferenceBitmapField::onVerify | Visibility check bypass | Verification control |
| CP-004 | ReferenceBitmapField::onVerify | Pixel comparison | Corruption detection |
| CP-005 | Canvas::verify | Texture load | Verification validity |
| CP-006 | Pool::allocate | Pool integrity check | Memory safety |
| CP-007 | Pool::deallocate | Double-free check | Memory corruption prevention |

### 4.2 Cyclomatic Complexity

| Function | Complexity | Risk Level |
|----------|------------|------------|
| Engine::Engine() | 4 | Low |
| Engine::getError() | 5 | Low |
| Pool::allocate() | 3 | Low |
| Pool::deallocate() | 5 | Low |
| Pool::checkPool() | 4 | Low |
| ReferenceBitmapField::onVerify() | 4 | Low |
| Canvas::verify() | 3 | Low |

All critical functions have cyclomatic complexity ≤ 10, which is acceptable for ASIL D.

---

## 5. Unreachable Code Analysis

### 5.1 Identified Defensive Code

| Location | Code | Status | Justification |
|----------|------|--------|---------------|
| Pool::allocate:210 | `else` after `checkPool()` fails | Reachable | Corruption detection path |
| Pool::deallocate:250 | `else` for corrupted pool | Reachable | Corruption detection path |
| Engine.cpp:42 | Empty `else` clause | Intentional | MISRA compliance placeholder |

### 5.2 Dead Code Assessment

No unreachable code identified in analyzed functions. All branches are reachable under specific conditions.

---

## 6. Summary

### 6.1 Control Flow Findings

- All critical functions have bounded complexity (≤ 5)
- No infinite loops possible in analyzed code
- All error paths terminate with appropriate error codes
- Multi-level validation in Pool operations

### 6.2 Data Flow Findings

- Clear data ownership throughout render pipeline
- Error propagation follows defined hierarchy
- No circular data dependencies
- Reference bitmap data integrity maintained through verification chain

### 6.3 Recommendations

1. **Pool::checkPool()** should be called before every pool operation (currently implemented)
2. **Error aggregation** correctly prioritizes engine-level errors
3. **Verification bypass** for invisible fields is intentional and documented

---

**End of Document**
