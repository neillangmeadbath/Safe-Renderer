# LSR-SVS-001: Software Verification Specification - Unit Testing

| Document ID | LSR-SVS-001 |
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
| LSR-SAD-001 | Software Architecture Description |
| LSR-DS-001 | Design Specification |
| ISO 26262:2018 Part 6 | Software development |

---

## 1. Introduction

### 1.1 Purpose

This document specifies the unit testing strategy for the Luxoft Safe Renderer to achieve ISO 26262 ASIL D compliance. It defines:
- Testing methodology and approach
- Coverage requirements
- Test case specifications
- Test environment requirements
- Verification methods

### 1.2 Scope

This specification covers unit testing for all certified LSR components:
- `engine/lsr` - Engine module
- `engine/database` - Database module
- `engine/display` - Display module
- `engine/framehandler` - FrameHandler module
- `engine/common` - Common utilities

### 1.3 ASIL D Unit Testing Requirements

Per ISO 26262-6 Table 9, ASIL D software unit testing requires:

| Method | ASIL D Requirement |
|--------|-------------------|
| Requirements-based testing | Highly Recommended (++) |
| Interface testing | Highly Recommended (++) |
| Fault injection testing | Highly Recommended (++) |
| Resource usage testing | Highly Recommended (++) |
| Back-to-back testing | Recommended (+) |

Coverage requirements per ISO 26262-6 Table 12:

| Coverage Metric | ASIL D Requirement |
|-----------------|-------------------|
| Statement Coverage | 100% |
| Branch Coverage | 100% |
| MC/DC Coverage | Highly Recommended (++) |

---

## 2. Test Strategy

### 2.1 Testing Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                    UNIT TESTING STRATEGY                         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Level 1: Requirements-Based Testing                     │    │
│  │  - Test each TSR                                         │    │
│  │  - Verify functional behavior                            │    │
│  │  - Cover normal and boundary conditions                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Level 2: Interface Testing                              │    │
│  │  - Test all public interfaces                            │    │
│  │  - Verify parameter validation                           │    │
│  │  - Test return values and error codes                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Level 3: Fault Injection Testing                        │    │
│  │  - Inject memory corruption                              │    │
│  │  - Simulate GIL failures                                 │    │
│  │  - Force error conditions                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Level 4: Coverage Analysis                              │    │
│  │  - Statement coverage (100%)                             │    │
│  │  - Branch coverage (100%)                                │    │
│  │  - MC/DC for safety-critical decisions                   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Test Framework

| Component | Tool/Framework |
|-----------|----------------|
| Test Framework | Google Test (gtest) |
| Mock Framework | Google Mock (gmock) |
| Coverage Analysis | gcov/lcov |
| Static Analysis | Coverity, cppcheck |

### 2.3 Test Organization

```
test/
├── engine/
│   ├── common/
│   │   ├── PoolTest.cpp
│   │   ├── AssertionTest.cpp
│   │   ├── LSRErrorCollectorTest.cpp
│   │   └── ...
│   ├── database/
│   │   ├── DatabaseTest.cpp
│   │   ├── AreaTest.cpp
│   │   └── ...
│   ├── display/
│   │   ├── DisplayTest.cpp
│   │   ├── TextureTest.cpp
│   │   └── ...
│   ├── framehandler/
│   │   ├── FrameHandlerTest.cpp
│   │   ├── WidgetTest.cpp
│   │   ├── BitmapFieldTest.cpp
│   │   ├── ReferenceBitmapFieldTest.cpp
│   │   └── ...
│   └── lsr/
│       ├── EngineTest.cpp
│       └── ...
└── mocks/
    ├── MockGIL.h
    ├── MockDatabase.h
    └── ...
```

---

## 3. Coverage Requirements

### 3.1 Statement Coverage (100% Required)

Every executable statement must be executed at least once.

| Module | Target | Measurement Method |
|--------|--------|-------------------|
| engine/common | 100% | gcov |
| engine/database | 100% | gcov |
| engine/display | 100% | gcov |
| engine/framehandler | 100% | gcov |
| engine/lsr | 100% | gcov |

### 3.2 Branch Coverage (100% Required)

Every branch in decision statements must be executed.

| Module | Target | Measurement Method |
|--------|--------|-------------------|
| engine/common | 100% | gcov |
| engine/database | 100% | gcov |
| engine/display | 100% | gcov |
| engine/framehandler | 100% | gcov |
| engine/lsr | 100% | gcov |

### 3.3 MC/DC Coverage (Safety-Critical Functions)

Modified Condition/Decision Coverage for safety-critical decisions:

| Function | Decision | MC/DC Required |
|----------|----------|----------------|
| Pool::checkPool() | Pool integrity check | Yes |
| Pool::allocate() | Allocation decision | Yes |
| Pool::deallocate() | Deallocation validity | Yes |
| ReferenceBitmapField::onVerify() | Verification result | Yes |
| Database validation | Configuration checks | Yes |

### 3.4 Coverage Exclusions

Justified exclusions from coverage requirements:

| Exclusion | Justification |
|-----------|---------------|
| Defensive code unreachable by design | Proven unreachable by static analysis |
| Platform-specific dead code | Conditional compilation |
| Third-party code (gtest/gmock) | Not in certification scope |

---

## 4. Test Categories

### 4.1 Normal Operation Tests

Tests verifying correct behavior under normal conditions.

#### 4.1.1 Pool Normal Operation

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-POOL-N-001 | Allocate single object | Return valid pointer, no error |
| TC-POOL-N-002 | Allocate maximum objects | All allocations succeed |
| TC-POOL-N-003 | Deallocate allocated object | Return LSR_NO_ENGINE_ERROR |
| TC-POOL-N-004 | isAllocated on allocated pointer | Return true |
| TC-POOL-N-005 | checkPool on valid pool | Return true |

#### 4.1.2 Database Normal Operation

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-DB-N-001 | Initialize with valid DDH | No error |
| TC-DB-N-002 | Get valid bitmap by ID | Return valid StaticBitmap |
| TC-DB-N-003 | Get panel by valid ID | Return valid panel |
| TC-DB-N-004 | getError after success | Return LSR_NO_ENGINE_ERROR |

#### 4.1.3 Engine Normal Operation

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-ENG-N-001 | Initialize with valid config | No error |
| TC-ENG-N-002 | Call render() | Return true |
| TC-ENG-N-003 | Call verify() | Return true |
| TC-ENG-N-004 | getError() after render | Return no error |

### 4.2 Boundary Tests

Tests at boundary values and limits.

#### 4.2.1 Pool Boundary Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-POOL-B-001 | Allocate when pool is full | Return NULL, LSR_POOL_IS_FULL |
| TC-POOL-B-002 | Deallocate with NULL pointer | Return LSR_POOL_INVALID_OBJECT |
| TC-POOL-B-003 | First allocation | Valid pointer |
| TC-POOL-B-004 | Last allocation | Valid pointer |

#### 4.2.2 Database Boundary Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-DB-B-001 | Get bitmap with ID 0 | Return valid or NULL per config |
| TC-DB-B-002 | Get bitmap with max valid ID | Return valid bitmap |
| TC-DB-B-003 | Get bitmap with max+1 ID | Return NULL |
| TC-DB-B-004 | Get bitmap with 0xFFFFFFFF | Return NULL |

#### 4.2.3 Area Boundary Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-AREA-B-001 | Area with x=0, y=0 | Valid area |
| TC-AREA-B-002 | Area with width=0 | Valid (empty) area |
| TC-AREA-B-003 | Area with max coordinates | Valid area |

### 4.3 Error Injection Tests

Tests injecting errors to verify detection.

#### 4.3.1 Memory Corruption Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-POOL-E-001 | Corrupt free marker (0xAA) | checkPool returns false |
| TC-POOL-E-002 | Corrupt busy marker (0x55) | isAllocated returns false |
| TC-POOL-E-003 | Double deallocate | Return LSR_POOL_DOUBLE_DELETE |
| TC-POOL-E-004 | Invalid pointer deallocate | Return LSR_POOL_INVALID_OBJECT |
| TC-POOL-E-005 | Corrupt free list (loop) | checkPool detects loop |
| TC-POOL-E-006 | Corrupt pool bounds | isAllocated returns false |

#### 4.3.2 Configuration Error Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-DB-E-001 | Invalid DDH magic number | Return LSR_DB_ERROR |
| TC-DB-E-002 | DDH version mismatch | Return LSR_DB_DDHBIN_VERSION_MISMATCH |
| TC-DB-E-003 | Empty DDH | Return LSR_DB_DDHBIN_EMPTY |
| TC-DB-E-004 | NULL bitmap data pointer | Return NULL, set error |

#### 4.3.3 GIL Error Injection Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-GIL-E-001 | gilCreateContext returns NULL | Engine reports error |
| TC-GIL-E-002 | gilCreateTexture returns NULL | LSR_ERROR_NO_TEXTURE |
| TC-GIL-E-003 | gilSetSurface returns false | Display error reported |
| TC-GIL-E-004 | gilVerify returns false | Verification error counted |

### 4.4 Interface Tests

Tests verifying interface contracts.

#### 4.4.1 Engine Interface Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-IF-ENG-001 | Engine constructor with NULL DDH | Assertion or error |
| TC-IF-ENG-002 | render() before init complete | Return false |
| TC-IF-ENG-003 | getError() type wrapper | Correct Error object |
| TC-IF-ENG-004 | Multiple sequential renders | All return true |

#### 4.4.2 Database Interface Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-IF-DB-001 | getBitmap with all valid IDs | All return valid |
| TC-IF-DB-002 | getPanel with all valid IDs | All return valid |
| TC-IF-DB-003 | Multiple getBitmap calls | Consistent results |

#### 4.4.3 Widget Interface Tests

| Test ID | Test Case | Expected Result |
|---------|-----------|-----------------|
| TC-IF-WGT-001 | setup() with valid database | No error |
| TC-IF-WGT-002 | onDraw() with valid canvas | Successful draw |
| TC-IF-WGT-003 | getError() after operation | Correct error state |

---

## 5. Module Test Specifications

### 5.1 Pool Module (engine/common/Pool.h)

#### 5.1.1 Test Environment

```cpp
class PoolTest : public ::testing::Test {
protected:
    static const size_t POOL_SIZE = 10;
    Pool<TestObject, POOL_SIZE> pool;
};
```

#### 5.1.2 Test Cases

| Test ID | TSR | Test Method | Pass Criteria |
|---------|-----|-------------|---------------|
| TC-POOL-001 | TSR-MS-001 | Allocate and check marker | Marker = 0x55 |
| TC-POOL-002 | TSR-MS-001 | Deallocate and check marker | Marker = 0xAA |
| TC-POOL-003 | TSR-MS-002 | checkPool on fresh pool | Returns true |
| TC-POOL-004 | TSR-MS-002 | checkPool after corruption | Returns false |
| TC-POOL-005 | TSR-MS-003 | Double deallocate detection | LSR_POOL_DOUBLE_DELETE |
| TC-POOL-006 | TSR-MS-004 | Pool exhaustion | LSR_POOL_IS_FULL |
| TC-POOL-007 | TSR-MS-005 | Invalid pointer check | LSR_POOL_INVALID_OBJECT |
| TC-POOL-008 | TSR-MS-005 | Bounds checking | False for out-of-bounds |
| TC-POOL-009 | TSR-MS-002 | Free list loop detection | Returns false |
| TC-POOL-010 | TSR-MS-006 | Compile-time size | Static verification |

#### 5.1.3 Fault Injection Strategy

```cpp
// Marker corruption injection
class PoolCorrupter {
public:
    static void corruptFreeMarker(void* poolStorage, size_t index);
    static void corruptBusyMarker(void* poolStorage, size_t index);
    static void createFreeListLoop(void* poolStorage, size_t index);
};
```

### 5.2 Database Module (engine/database)

#### 5.2.1 Test Environment

```cpp
class DatabaseTest : public ::testing::Test {
protected:
    // Valid test DDH data
    static const DDHType validDDH;
    // Corrupted DDH variants
    static const DDHType invalidMagicDDH;
    static const DDHType versionMismatchDDH;
};
```

#### 5.2.2 Test Cases

| Test ID | TSR | Test Method | Pass Criteria |
|---------|-----|-------------|---------------|
| TC-DB-001 | TSR-DD-001 | Valid DDH initialization | No error |
| TC-DB-002 | TSR-DD-001 | Invalid magic number | LSR_DB_ERROR |
| TC-DB-003 | TSR-DD-003 | Version mismatch | LSR_DB_DDHBIN_VERSION_MISMATCH |
| TC-DB-004 | TSR-DD-002 | NULL structure pointers | Error detected |
| TC-DB-005 | TSR-DD-006 | Valid bitmap ID | Returns bitmap |
| TC-DB-006 | TSR-DD-006 | Invalid bitmap ID | Returns NULL |
| TC-DB-007 | TSR-DD-004 | Bitmap data validation | Non-NULL data |
| TC-DB-008 | TSR-DD-005 | Bitmap dimension check | Valid dimensions |

### 5.3 Display Module (engine/display)

#### 5.3.1 Test Environment

```cpp
class DisplayTest : public ::testing::Test {
protected:
    MockGIL mockGIL;
    // Setup mock expectations
    void SetUp() override;
};
```

#### 5.3.2 Test Cases

| Test ID | TSR | Test Method | Pass Criteria |
|---------|-----|-------------|---------------|
| TC-DISP-001 | TSR-IN-002 | Successful init | Context created |
| TC-DISP-002 | TSR-IN-002 | Init with GIL failure | Error reported |
| TC-DISP-003 | TSR-DD-008 | Texture loading | Texture valid |
| TC-DISP-004 | TSR-DD-008 | Texture loading failure | LSR_ERROR_NO_TEXTURE |
| TC-DISP-005 | TSR-TI-003 | Swap buffers | Success |

### 5.4 FrameHandler Module (engine/framehandler)

#### 5.4.1 Test Environment

```cpp
class FrameHandlerTest : public ::testing::Test {
protected:
    MockDatabase mockDB;
    MockDisplayManager mockDisplay;
    MockIHMI mockIHMI;
};
```

#### 5.4.2 Widget Test Cases

| Test ID | TSR | Test Method | Pass Criteria |
|---------|-----|-------------|---------------|
| TC-WGT-001 | TSR-AV-006 | Widget setup | No error |
| TC-WGT-002 | TSR-DD-007 | BitmapField draw | Correct position |
| TC-WGT-003 | TSR-VE-001 | ReferenceBitmapField verify | gilVerify called |
| TC-WGT-004 | TSR-VE-004 | Verification error count | Counter increments |
| TC-WGT-005 | TSR-VE-005 | Invisible skip verify | No gilVerify call |
| TC-WGT-006 | TSR-AV-002 | Render completion | Returns true |

### 5.5 Engine Module (engine/lsr)

#### 5.5.1 Test Environment

```cpp
class EngineTest : public ::testing::Test {
protected:
    MockIHMI mockIHMI;
    static const DDHType testDDH;
};
```

#### 5.5.2 Test Cases

| Test ID | TSR | Test Method | Pass Criteria |
|---------|-----|-------------|---------------|
| TC-ENG-001 | TSR-IN-001 | Valid initialization | getError() = 0 |
| TC-ENG-002 | TSR-IN-003 | Render after init fail | Returns false |
| TC-ENG-003 | TSR-AV-001 | Successful render | Returns true |
| TC-ENG-004 | TSR-AV-003 | Error aggregation | Highest severity |
| TC-ENG-005 | TSR-AV-005 | Error persistence | State retained |

---

## 6. Verification Methods

### 6.1 Requirements-Based Testing

Each TSR must have at least one test case:

| TSR | Test Cases | Coverage |
|-----|------------|----------|
| TSR-DD-001 | TC-DB-001, TC-DB-002 | Complete |
| TSR-DD-002 | TC-DB-004 | Complete |
| TSR-DD-003 | TC-DB-003 | Complete |
| TSR-MS-001 | TC-POOL-001, TC-POOL-002, TC-POOL-004 | Complete |
| TSR-MS-002 | TC-POOL-003, TC-POOL-004, TC-POOL-009 | Complete |
| TSR-MS-003 | TC-POOL-005 | Complete |
| ... | ... | ... |

### 6.2 Interface Testing

For each public interface:

```cpp
// Example interface test
TEST_F(EngineTest, RenderInterface) {
    // Pre-condition
    ASSERT_TRUE(engine.getError().getValue() == LSR_NO_ENGINE_ERROR);

    // Execute
    bool result = engine.render();

    // Post-condition
    EXPECT_TRUE(result);
    EXPECT_EQ(engine.getError().getValue(), LSR_NO_ENGINE_ERROR);
}
```

### 6.3 Fault Injection Testing

| Fault Type | Injection Method | Verification |
|------------|------------------|--------------|
| Memory corruption | Direct memory write | Detection verified |
| GIL failure | Mock return values | Error handling verified |
| NULL pointers | Pass NULL arguments | Graceful handling |
| Invalid IDs | Out-of-range values | Bounds checking |

### 6.4 Resource Usage Testing

| Resource | Test Method | Pass Criteria |
|----------|-------------|---------------|
| Stack depth | Static analysis | Within limits |
| Pool capacity | Exhaust and recover | No crash |
| Timing | Measure execution | Within budget |

---

## 7. Test Environment

### 7.1 Build Configuration

```cmake
# Test build configuration
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage -g -O0")
set(UNIT_TESTS ON)
```

### 7.2 Mock Objects

| Mock | Purpose | Implementation |
|------|---------|----------------|
| MockGIL | Simulate GIL functions | gmock expectations |
| MockDatabase | Simulate database access | gmock expectations |
| MockIHMI | Simulate HMI interface | gmock expectations |
| MockCanvas | Simulate canvas operations | gmock expectations |

### 7.3 Test Data

| Data | Description | Location |
|------|-------------|----------|
| Valid DDH | Complete valid configuration | test/database/Telltales |
| Invalid DDH variants | Corruption test data | test/data/invalid/ |
| Reference bitmaps | Verification test images | test/images/ |

---

## 8. Test Execution

### 8.1 Test Execution Order

1. Common module tests (foundation)
2. Database module tests (data layer)
3. Display module tests (rendering infrastructure)
4. FrameHandler module tests (widget hierarchy)
5. Engine module tests (integration)

### 8.2 Test Commands

```bash
# Build tests
cmake -DUNIT_TESTS=ON ..
make

# Run all tests
ctest --output-on-failure

# Run with coverage
./run_tests
lcov --capture --directory . --output-file coverage.info
genhtml coverage.info --output-directory coverage_report

# Run specific module
./common_test
./database_test
./display_test
./framehandler_test
./engine_test
```

### 8.3 Pass/Fail Criteria

| Criteria | Requirement |
|----------|-------------|
| All tests pass | 100% pass rate |
| Statement coverage | ≥100% |
| Branch coverage | ≥100% |
| MC/DC coverage | ≥100% for safety decisions |
| No memory leaks | Valgrind clean |

---

## 9. Test Traceability

### 9.1 TSR to Test Case Matrix

| TSR ID | Test Cases | Status |
|--------|------------|--------|
| TSR-DD-001 | TC-DB-001, TC-DB-002 | Specified |
| TSR-DD-002 | TC-DB-004 | Specified |
| TSR-DD-003 | TC-DB-003 | Specified |
| TSR-DD-004 | TC-DB-007 | Specified |
| TSR-DD-005 | TC-DB-008 | Specified |
| TSR-DD-006 | TC-DB-005, TC-DB-006 | Specified |
| TSR-DD-007 | TC-WGT-002 | Specified |
| TSR-DD-008 | TC-DISP-003, TC-DISP-004 | Specified |
| TSR-AV-001 | TC-ENG-003 | Specified |
| TSR-AV-002 | TC-WGT-006 | Specified |
| TSR-AV-003 | TC-ENG-004 | Specified |
| TSR-AV-004 | TC-ENG-003, TC-DB-004 | Specified |
| TSR-AV-005 | TC-ENG-005 | Specified |
| TSR-AV-006 | TC-WGT-001 | Specified |
| TSR-MS-001 | TC-POOL-001, TC-POOL-002, TC-POOL-004 | Specified |
| TSR-MS-002 | TC-POOL-003, TC-POOL-004, TC-POOL-009 | Specified |
| TSR-MS-003 | TC-POOL-005 | Specified |
| TSR-MS-004 | TC-POOL-006 | Specified |
| TSR-MS-005 | TC-POOL-007, TC-POOL-008 | Specified |
| TSR-MS-006 | TC-POOL-010 | Specified |
| TSR-VE-001 | TC-WGT-003 | Specified |
| TSR-VE-002 | TC-GIL-E-004 | Specified |
| TSR-VE-003 | TC-WGT-003 | Specified |
| TSR-VE-004 | TC-WGT-004 | Specified |
| TSR-VE-005 | TC-WGT-005 | Specified |
| TSR-ER-001 | TC-ENG-004 | Specified |
| TSR-ER-002 | Code review | Specified |
| TSR-ER-003 | Code review | Specified |
| TSR-ER-004 | TC-ASSERT-001 | Specified |
| TSR-IN-001 | TC-DB-001, TC-ENG-001 | Specified |
| TSR-IN-002 | TC-DISP-001, TC-DISP-002 | Specified |
| TSR-IN-003 | TC-ENG-002 | Specified |
| TSR-FI-001 | TC-FI-001 | Specified |
| TSR-FI-002 | TC-FI-002 | Specified |
| TSR-FI-003 | TC-FI-003 | Specified |

---

## 10. Appendices

### Appendix A: Test Case Template

```cpp
/**
 * @test TC-XXX-NNN
 * @brief Brief description
 * @req TSR-XX-NNN
 * @pre Preconditions
 * @steps
 *   1. Step one
 *   2. Step two
 * @expected Expected result
 */
TEST_F(TestClass, TestName) {
    // Setup

    // Execute

    // Verify
}
```

### Appendix B: Coverage Report Template

```
Module: engine/common
================================================================================
File                                          Line   Branch  MC/DC
--------------------------------------------------------------------------------
Pool.h                                        100%   100%    100%
Assertion.h                                   100%   100%    N/A
LSRErrorCollector.h                           100%   100%    N/A
--------------------------------------------------------------------------------
Total                                         100%   100%    100%
```

### Appendix C: Fault Injection Techniques

| Technique | Implementation | Target |
|-----------|----------------|--------|
| Memory write | Direct pointer manipulation | Pool markers |
| Return value | Mock configuration | GIL functions |
| Parameter | Invalid arguments | Interface methods |
| State | Pre-corrupt data | Configuration data |

---

**End of Document**
