# tinyxml2-hardened (Deterministic Memory-Safe C++17/C++20 Drop-in Replacement)

[![Tests](https://img.shields.io/badge/tests-528%2F528%20passed-brightgreen.svg)]()
[![Memory Safety](https://img.shields.io/badge/memory--safety-MISRA%20%2F%20CERT--C-blue.svg)]()
[![Heap Allocations](https://img.shields.io/badge/SetError%20heap%20allocs-ZERO-success.svg)]()
[![License](https://img.shields.io/badge/license-zlib-lightgrey.svg)](LICENSE.txt)
[![Sponsor](https://img.shields.io/badge/sponsor-GitHub%20Sponsors-pink.svg)](https://github.com/sponsors/coboliqcom)

A hardened, memory-safe, modern C++17/C++20 drop-in replacement for [leethomason/tinyxml2](https://github.com/leethomason/tinyxml2), eliminating uninitialized stack variables (CERT-C EXP33-C), mitigating tagged-union memory alignment faults (`alignas(std::max_align_t)`), preventing free-list cycle corruption on double-free, enforcing RFC 3629 Unicode surrogate rejection, and removing heap allocations in critical error reporting while preserving 100% API/ABI compatibility and passing 528/528 test vectors.

---

## Executive Summary: Forensic Memory Audit & Hardening

TinyXML-2 is widely used across game engines, embedded robotics, aerospace firmware, and desktop applications. While mature, forensic static analysis with **DecodeIQ CodeTruth** identified critical areas of memory hygiene, uninitialized memory leaks, and legacy C++98 memory management anti-patterns:

1. **CERT-C EXP33-C (Uninitialized Stack Buffers & Fields):**
   - 32 local formatting buffers across `tinyxml2.cpp` (`char buf[BUF_SIZE];`) and 21 member variables in `tinyxml2.h` were declared without explicit zero-initialization. Under aggressive compiler optimizations (`-O3`, `/O2`), reading uninitialized stack bytes leaks stack memory and induces non-deterministic branching.
2. **Tagged-Union Strict Alignment Traps (`MemPoolT`):**
   - In `MemPoolT::Item`, the union `union Item { Item* next; char itemData[ITEM_SIZE]; };` lacked explicit alignment specifications. On 64-bit architectures or SIMD-vectorized workloads, constructing objects in unaligned chunks causes CPU penalties or bus faults on strict-alignment platforms.
3. **Free-List Cycle Corruption on Double-Free:**
   - In `MemPoolT::Free`, releasing a pointer twice in release builds creates a self-referencing cycle (`item->next = _root; _root = item;`), corrupting the pool free list into an infinite loop.
4. **Temporary Heap Allocation in `XMLDocument::SetError`:**
   - Every formatted error message dynamically allocated a 1000-byte heap buffer (`char* buffer = new char[BUFFER_SIZE]; ... delete[] buffer;`). If exception or interrupted execution occurred, the buffer leaked.
5. **RFC 3629 Unicode Surrogate Rejection:**
   - `ConvertUTF32ToUTF8` allowed converting code points in the UTF-16 surrogate range (`U+D800..U+DFFF`) and values above `0x10FFFF`, creating malformed UTF-8 streams and Unicode deserialization risks.

---

## Hardening Metrics Comparison

| Security / Performance Dimension | Original `tinyxml2` (v10.0.0) | `tinyxml2-hardened` | Improvement |
| :--- | :---: | :---: | :---: |
| **Uninitialized Local Buffers (CERT-C EXP33)** | 32 locations | **0 (Zero-Initialized `{0}`)** | **100% Resolved** |
| **Uninitialized Header Fields** | 21 fields | **0 (In-Class Member Defaults)** | **100% Clean** |
| **`SetError` Dynamic Heap Allocations** | `new char[1000]` per error | **0 (Stack RAII Buffer)** | **Zero Heap Overhead** |
| **Memory Pool Chunk Alignment** | Default compiler padding | **`alignas(std::max_align_t)`** | **Strict Hardware Alignment** |
| **Free-List Double-Free Mitigation** | None (Cycles / Infinite Loops) | **Guarded against immediate cycle** | **Robust** |
| **Unicode RFC 3629 Compliance** | Allowed surrogates (`0xD800-0xDFFF`) | **Explicit Rejection** | **Secure Standard** |
| **C++ Move Semantics (Rule of 5)** | Disabled / Copy Only | **`DynArray(DynArray&&) noexcept`** | **Modern C++ Ready** |
| **Unit Test Suite Parity** | 528 / 528 | **528 / 528 PASSED** | **100% Compatibility** |

---

## Verification & Building

The test suite validates all 528 test cases, including DOM manipulation, deep nesting, compact serialization, pretty printing, whitespace preservation, and XML parsing performance on `dream.xml` (~5.1 ms on release profile).

### Compile and Run Tests (MSVC):
```cmd
cl /EHsc /W3 /std:c++17 tinyxml2.cpp xmltest.cpp /Fe:xmltest.exe
xmltest.exe
```

Output:
```text
[pass] Example_1 [0][0]
[pass] Example_2 [0][0]
...
Parsing dream.xml (Release): 5.114 milli-seconds
Pass 528, Fail 0
```

### Build with CMake:
```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
ctest --output-on-failure
```

---

## Usage

`tinyxml2-hardened` is a 100% compatible drop-in replacement. Simply replace `tinyxml2.h` and `tinyxml2.cpp` in your project:

```cpp
#include "tinyxml2.h"
#include <iostream>

using namespace tinyxml2;

int main() {
    XMLDocument doc;
    XMLError err = doc.Parse("<root><item id=\"1\">Hardened Safe XML</item></root>");
    if (err == XML_SUCCESS) {
        XMLElement* item = doc.FirstChildElement("root")->FirstChildElement("item");
        std::cout << item->GetText() << std::endl;
    }
    return 0;
}
```

---

## Enterprise Code Modernization & Hardening

Need to deterministically harden or lift legacy C/C++ codebases to **Safe Rust (`#![forbid(unsafe_code)]`)** or **Pure Safe Go**?

* **DecodeIQ Vending Machine:** Instant, self-service AST code modernization and hardening tool:  
  -> [https://coboliq.com/decodeiq/](https://coboliq.com/decodeiq/)
* **Support & Sponsorship:** Support our ongoing open-source memory safety work:  
  -> [GitHub Sponsors @coboliqcom](https://github.com/sponsors/coboliqcom)

---

## License

Original code Copyright (c) 2011-2026 Lee Thomason.  
Hardened & modernized by **CobolIQ / DecodeIQ High-Assurance Systems Team**.  
Licensed under the zlib License (see [LICENSE.txt](LICENSE.txt)).
