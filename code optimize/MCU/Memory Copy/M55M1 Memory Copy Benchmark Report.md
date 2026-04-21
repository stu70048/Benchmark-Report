# M55M1 Memory Copy Benchmark Report

**Date**: 2026/04/21  
**Platform**: Nuvoton M55M1 (Arm Cortex-M55)  
**System Clock**: 220 MHz  
**Copy Size**: 512 KB (524,288 bytes)  
**Test Rounds**: 512  
**Compiler**: ARMCLANG (Keil MDK / CMSIS-Toolbox)

---

## 1. Objective

Compare the performance of different C loop styles for memory copy on Cortex-M55, covering three usage scenarios:

| Scenario        | Description                            | Real-World Example                        |
| --------------- | -------------------------------------- | ----------------------------------------- |
| **Memory Copy** | Both src and dst are RAM buffers       | General data transfer                     |
| **RegWrite**    | src is RAM, dst is a fixed HW register | Write to fixed register (e.g. HSUSBD EPDAT) |
| **RegRead**     | src is a fixed HW register, dst is RAM | Read from fixed register (e.g. HSUSBD EPDAT) |

---

## 2. Program Architecture and Test Functions

### 2.1 Overall Program Structure

```text
main()
 +-- SYS_Init()                    // System init: PLL0 220MHz, UART
 +-- Benchmark_TimerInit()         // SysTick 24-bit cycle counter init
 +-- FillSourcePattern()           // Fill src buffer with 0x00~0xFF pattern
 |
 +-- Build BenchItem_t asBench[]   // Array of all benchmark items
 |    +-- ByteCopy / WordCopy      // Baseline tests
 |    +-- HybridV1 ~ V9           // Memory Copy variants (9 loop styles)
 |    +-- RegWrite V1~V9          // Fixed Reg Write variants (7 loop styles)
 |    +-- RegRead 5 variants      // Fixed Reg Read variants (5 loop styles)
 |
 +-- RunAllBenchmarks()            // Execute all tests + memcmp verification
 |    +-- RunCopyBenchmark()       // Per-item: 512 rounds, SysTick timing
 |         +-- __disable_irq()     // Disable IRQ to avoid interference
 |         +-- SysTick->VAL        // Capture start tick
 |         +-- pfnCopy()           // Call test function
 |         +-- SysTick->VAL        // Capture end tick
 |         +-- __enable_irq()      // Restore IRQ
 |
 +-- PrintResults()                // Print cycles/throughput + comparison table
```

**Key Design Decisions**:

- `volatile` pointers prevent the compiler from optimizing away the copy loops
- SysTick 24-bit timer: max single measurement ~76ms @220MHz (512KB takes ~9ms, well within range)
- IRQ disabled (`__disable_irq`) during measurement to avoid interference
- Unified `CopyFunc_t` function pointer interface; each test function has a corresponding Wrapper

### 2.2 Global Resources

```c
static uint8_t  g_au8Src[512KB]       __attribute__((aligned(4)));  // Source buffer
static uint8_t  g_au8DstPool[512KB+4] __attribute__((aligned(4)));  // Destination buffer
static volatile uint32_t g_u32FakeReg;  // Simulated HW register (RegWrite sink / RegRead source)
```

### 2.3 Test Functions Overview

#### Memory Copy Variants

All Hybrid variants follow a 3-phase pattern: **head byte-copy (alignment) -> word-copy body -> tail byte-copy**.

| Function   | Loop Style                                           | Description                     |
| ---------- | ---------------------------------------------------- | ------------------------------- |
| `ByteCopy` | `for (i=0; i<len; i++) dst[i]=src[i]` (uint8)       | Byte-by-byte copy, baseline     |
| `WordCopy` | `for (i=0; i<cnt; i++) dst[i]=src[i]` (uint32)      | Word copy (requires alignment)  |
| `HybridV1` | `outpw(ptr, val); i+=4; ptr+=4;`                    | outpw macro + 3 increments      |
| `HybridV2` | `pu32D[j] = pu32S[j]`                               | **Indexed access**              |
| `HybridV3` | `outpw(ptr+j*4, *(src+j*4))`                        | outpw + index address calc      |
| `HybridV4` | `outpw(pu32D+j, *(pu32S+j))`                        | outpw + word pointer index      |
| `HybridV5` | `w = *pu32S++; *pu32D++ = w;`                        | Pointer increment + temp var    |
| `HybridV6` | `pu32D[j]=pu32S[j]` x4 unroll                       | V2 + 4x manual unrolling       |
| `HybridV7` | `while(countdown > 3) { *d++=*s++; countdown-=4; }` | Countdown (TinyUSB-like style)  |
| `HybridV8` | `while(ptr < end) *d++=*s++;`                        | Pointer comparison termination  |
| `HybridV9` | `while(count--) *d++=*s++;`                          | Count decrement + pointer inc   |

#### RegWrite Variants (Write to Fixed Register)

Write target is always `g_u32FakeReg` (simulated HW register); only src pointer advances.  
V1/V2/V5/V6/V7/V8/V9 loop styles correspond to their HybridCopy counterparts.

```c
// Key difference: dst is fixed, only src advances
for (j = 0u; j < u32Words; j++)
    g_u32FakeReg = pu32S[j];       // V2: indexed read -> fixed reg write
```

#### RegRead Variants (Read from Fixed Register)

Read source is always `g_u32FakeReg`; only dst pointer advances.

| Function           | Loop Style                                 |
| ------------------ | ------------------------------------------ |
| `RegReadIndexed`   | `for(j<n) pu32D[j] = g_u32FakeReg`        |
| `RegReadForPtrInc` | `for(j<n) *pu32D++ = g_u32FakeReg`         |
| `RegReadPtrInc`    | `for(n>0;n--) *pu32D++ = g_u32FakeReg`     |
| `RegReadPtrEnd`    | `while(ptr < end) *pu32D++ = g_u32FakeReg` |
| `RegReadPtrEndV2`  | `while(n--) *pu32D++ = g_u32FakeReg`        |

---

## 3. Benchmark Results

### 3.1 Cache ON (D-Cache Enabled)

| Function                | Avg Cycles  | Time/Round (us) | Throughput (MB/s) | vs ByteCopy |
| ----------------------- | ----------: | --------------: | ----------------: | ----------: |
| ByteCopy (uint8)        |   1,049,964 |           4,772 |            104.77 |       1.00x |
| **WordCopy (uint32)**   | **296,300** |       **1,346** |        **371.47** |   **3.54x** |
| HybridV1 outpw+3inc     |     787,829 |           3,581 |            139.62 |       1.33x |
| **HybridV2 indexed**    | **296,316** |       **1,346** |        **371.47** |   **3.54x** |
| **HybridV3 outpw+idx**  | **296,316** |       **1,346** |        **371.47** |   **3.54x** |
| **HybridV4 outpw+wptr** | **296,316** |       **1,346** |        **371.47** |   **3.54x** |
| HybridV5 for+\*p++      |     361,850 |           1,644 |            304.13 |       2.90x |
| HybridV6 idx+unroll     |     329,081 |           1,495 |            334.44 |       3.19x |
| HybridV7 countdown      |     657,662 |           2,989 |            167.28 |       1.59x |
| HybridV8 ptr\<end       |     722,291 |           3,283 |            152.29 |       1.45x |
| HybridV9 while(n--)     |     394,609 |           1,793 |            278.86 |       2.66x |
| RegWr V1 outpw+3inc     |     788,363 |           3,583 |            139.54 |       1.33x |
| **RegWr V2 indexed**    | **266,097** |       **1,209** |        **413.56** |   **3.94x** |
| RegWr V5 for+\*p++      |     298,327 |           1,356 |            368.73 |       3.51x |
| **RegWr V6 idx+unroll** | **266,097** |       **1,209** |        **413.56** |   **3.94x** |
| RegWr V7 countdown      |     591,665 |           2,689 |            185.94 |       1.77x |
| RegWr V8 ptr\<end       |     560,849 |           2,549 |            196.15 |       1.87x |
| RegWr V9 while(n--)     |     298,321 |           1,356 |            368.73 |       3.51x |
| **RegRd indexed**       | **294,959** |       **1,340** |        **373.13** |   **3.55x** |
| RegRd for+\*dst++       |     327,726 |           1,489 |            335.79 |       3.20x |
| RegRd while(n--)        |     327,718 |           1,489 |            335.79 |       3.20x |
| RegRd ptr\<end          |     622,627 |           2,830 |            176.67 |       1.68x |
| RegRd for(n>0;n--)      |     327,719 |           1,489 |            335.79 |       3.20x |

### 3.2 Cache OFF (D-Cache Disabled)

| Function                | Avg Cycles    | Time/Round (us) | Throughput (MB/s) | vs ByteCopy |
| ----------------------- | ------------: | --------------: | ----------------: | ----------: |
| ByteCopy (uint8)        |     4,718,622 |          21,448 |             23.31 |       1.00x |
| WordCopy (uint32)       |     1,212,446 |           5,511 |             90.72 |       3.89x |
| HybridV1 outpw+3inc     |     1,703,975 |           7,745 |             64.55 |       2.76x |
| HybridV2 indexed        |     1,212,462 |           5,511 |             90.72 |       3.89x |
| HybridV3 outpw+idx      |     1,212,462 |           5,511 |             90.72 |       3.89x |
| HybridV4 outpw+wptr     |     1,212,462 |           5,511 |             90.72 |       3.89x |
| HybridV5 for+\*p++      |     1,277,996 |           5,809 |             86.07 |       3.69x |
| HybridV6 idx+unroll     |     1,245,227 |           5,660 |             88.33 |       3.78x |
| HybridV7 countdown      |     1,572,895 |           7,149 |             69.93 |       2.99x |
| HybridV8 ptr\<end       |     1,638,437 |           7,447 |             67.14 |       2.87x |
| HybridV9 while(n--)     |     1,310,755 |           5,957 |             83.93 |       3.59x |
| RegWr V1 outpw+3inc     |     1,703,994 |           7,745 |             64.55 |       2.76x |
| **RegWr V2 indexed**    | **1,179,712** |       **5,362** |         **93.24** |   **3.99x** |
| RegWr V5 for+\*p++      |     1,212,475 |           5,511 |             90.72 |       3.89x |
| **RegWr V6 idx+unroll** | **1,179,708** |       **5,362** |         **93.24** |   **3.99x** |
| RegWr V7 countdown      |     1,507,389 |           6,851 |             72.98 |       3.13x |
| RegWr V8 ptr\<end       |     1,474,625 |           6,702 |             74.60 |       3.19x |
| RegWr V9 while(n--)     |     1,212,469 |           5,511 |             90.72 |       3.89x |
| RegRd indexed            |     1,212,462 |           5,511 |             90.72 |       3.89x |
| RegRd for+\*dst++       |     1,245,229 |           5,660 |             88.33 |       3.78x |
| RegRd while(n--)        |     1,245,221 |           5,660 |             88.33 |       3.78x |
| RegRd ptr\<end          |     1,540,129 |           7,000 |             71.42 |       3.06x |
| RegRd for(n>0;n--)      |     1,245,221 |           5,660 |             88.33 |       3.78x |

---

## 4. Analysis and Observations

### 4.1 Loop Style Performance Ranking (Cache ON, Fastest to Slowest)

| Rank | Category                             | Representative Function | Avg Cycles | Throughput   |
| :--: | ------------------------------------ | ----------------------- | ---------: | -----------: |
| 1st  | Indexed access                       | RegWr V2 / V6          |    266,097 |  413.56 MB/s |
| 1st  | Indexed access                       | RegRd Indexed           |    294,959 |  373.13 MB/s |
| 1st  | Indexed access                       | HybridV2 / V3 / V4     |    296,316 |  371.47 MB/s |
| 2nd  | `for+*p++` / `while(n--)` + ptr     | RegWr V5 / V9          |    298,321 |  368.73 MB/s |
| 2nd  | `for+*dst++` / `while(n--)` + ptr   | RegRd for/while        |    327,718 |  335.79 MB/s |
| 2nd  | Indexed + 4x unroll                 | HybridV6               |    329,081 |  334.44 MB/s |
| 2nd  | `for+*p++` + temp                   | HybridV5               |    361,850 |  304.13 MB/s |
| 2nd  | `while(count--)` + ptr++            | HybridV9               |    394,609 |  278.86 MB/s |
| 3rd  | `while(ptr < end)`                  | RegWr V8               |    560,849 |  196.15 MB/s |
| 3rd  | Countdown (`while n>3, n-=4`)       | RegWr V7               |    591,665 |  185.94 MB/s |
| 3rd  | `while(ptr < end)`                  | RegRd ptr\<end         |    622,627 |  176.67 MB/s |
| 3rd  | Countdown                           | HybridV7               |    657,662 |  167.28 MB/s |
| 3rd  | `while(ptr < end)`                  | HybridV8               |    722,291 |  152.29 MB/s |
| 3rd  | `outpw` + 3 ptr++                   | HybridV1 / RegWr V1   |    788,363 |  139.54 MB/s |
| 4th  | Byte-by-byte (uint8)                | ByteCopy               |  1,049,964 |  104.77 MB/s |

### 4.2 Key Findings

#### Finding 1: Indexed Access `arr[j]` Is the Optimal Loop Pattern

- Memory Copy: HybridV2/V3/V4 are **identical** to pure WordCopy (296,316 cycles)
- RegWrite: V2/V6 are the fastest (266,097 cycles), 11% faster than Memory Copy
- RegRead: Indexed is the fastest (294,959 cycles)
- **Root Cause**: `for (j=0; j<N; j++)` is the counting loop pattern most easily recognized by the compiler, enabling full utilization of the Cortex-M55 **Low Overhead Branch (LOB)** hardware loop mechanism, reducing branch overhead to zero

#### Finding 2: `while(countdown > 3)` and `while(ptr < end)` Are the Slowest

| Style        | Cache ON Cycles | Slowdown vs Indexed |
| ------------ | --------------: | ------------------: |
| V7 countdown |         657,662 |               2.22x |
| V8 ptr\<end  |         722,291 |               2.44x |

- **Root Cause**: `countdown -= 4` modifies the counter by a non-unity step; the compiler cannot convert it to a LOB loop
- `ptr < end` uses pointer comparison rather than count decrement, also preventing LOB utilization

#### Finding 3: `outpw()` + Multiple Pointer Increments (V1) Is Consistently the Slowest Word-Copy

- V1 executes `i += 4`, `pu8Dst += 4`, `pu8Src += 4` per iteration — three increment operations
- Excessive instruction count; `outpw` macro may introduce additional indirect access
- Cache ON: 788K vs indexed 296K -> **2.66x slower**

#### Finding 4: Manual 4x Unroll (V6) Provides No Additional Benefit

| Scenario    | Indexed (V2) | Unrolled (V6) | Difference          |
| ----------- | -----------: | ------------: | ------------------- |
| Memory Copy |      296,316 |       329,081 | V6 is **11% slower** |
| RegWrite    |      266,097 |       266,097 | **Identical**       |

- Under `-O2` optimization, the compiler already auto-unrolls indexed loops
- Manual unrolling may interfere with the compiler's register allocation strategy

#### Finding 5: Cache Has a Massive Impact on Performance

| Function |    Cache ON |   Cache OFF | Degradation |
| -------- | ----------: | ----------: | ----------: |
| ByteCopy | 104.77 MB/s |  23.31 MB/s |   **4.49x** |
| WordCopy | 371.47 MB/s |  90.72 MB/s |   **4.09x** |
| RegWr V2 | 413.56 MB/s |  93.24 MB/s |   **4.43x** |

- With Cache OFF, the performance gap between word-copy variants narrows (bus latency dominates; loop overhead becomes a smaller fraction)
- The countdown/ptr\<end disadvantage shrinks from 2.2x to ~1.3x with Cache OFF

#### Finding 6: `*p++` Style (V5/V9) Is Slightly Slower Than Indexed

- Cache ON: V5 361K vs V2 296K (**22% slower**)
- `*p++` requires a post-increment micro-operation; indexed `arr[j]` can use the more efficient base+offset addressing mode
- However, in the RegWrite scenario the gap is smaller (V5: 298K vs V2: 266K, 12% difference)

---

## 5. Recommendations

### Best Practice: Use Indexed Access + Counting Loop

```c
/* -- Memory Copy (Recommended: HybridV2 style) -- */
volatile uint32_t       *pu32D = (volatile uint32_t *)pu8Dst;
volatile const uint32_t *pu32S = (volatile const uint32_t *)pu8Src;
uint32_t j, u32Words = u32Len / 4u;

for (j = 0u; j < u32Words; j++)
    pu32D[j] = pu32S[j];           /* indexed: fastest, LOB-friendly */
```

```c
/* -- Write to Fixed Register (Recommended: RegWriteV2 style) -- */
/* Use case: HSUSBD EPDAT IN, SPI FIFO write, etc. */
volatile const uint32_t *pu32S = (volatile const uint32_t *)pu8Buf;
uint32_t j, u32Words = u32Len / 4u;

for (j = 0u; j < u32Words; j++)
    FIXED_REG = pu32S[j];          /* indexed read from RAM, write to fixed reg */
```

```c
/* -- Read from Fixed Register (Recommended: RegReadIndexed style) -- */
/* Use case: HSUSBD EPDAT OUT, SPI FIFO read, etc. */
volatile uint32_t *pu32D = (volatile uint32_t *)pu8Buf;
uint32_t j, u32Words = u32Len / 4u;

for (j = 0u; j < u32Words; j++)
    pu32D[j] = FIXED_REG;          /* read from fixed reg, indexed write to RAM */
```

### Why This Is Recommended

| Advantage                  | Explanation                                                  |
| -------------------------- | ------------------------------------------------------------ |
| **Best performance**       | 371~413 MB/s with Cache ON, fastest among all variants       |
| **LOB compatible**         | `for (j=0; j<N; j++)` perfectly matches Cortex-M55 HW loops |
| **Best in both Cache ON/OFF** | Optimal regardless of cache configuration                 |
| **Compiler friendly**      | Standard counting loop; compiler can auto-vectorize/unroll   |
| **Readable**               | Clear semantics, easy to maintain and code review            |
| **No manual unroll needed** | Compiler handles it automatically under -O2                 |

### Patterns to Avoid

| Pattern                                          | Problem                            | Perf Loss (Cache ON) |
| ------------------------------------------------ | ---------------------------------- | -------------------: |
| `while(countdown > 3) { ...; countdown -= 4; }`  | Non-standard counting loop, no LOB |        **2.22x slower** |
| `while(ptr < end) *d++ = *s++;`                   | Pointer comparison, no LOB         |        **2.44x slower** |
| `outpw(ptr, val); i+=4; ptr+=4;`                  | Too many instructions per iteration |        **2.66x slower** |
| Byte-by-byte copy (uint8)                        | Does not utilize 32-bit bus width  |        **3.54x slower** |

### D-Cache Usage Notes

- **Always enable D-Cache**: word copy throughput improves from ~91 MB/s to ~371 MB/s (**4.1x**)
- For HW register regions (e.g. HSUSBD EPDAT), configure MPU as **Device / Non-cacheable** to avoid cache coherency issues
- For general RAM buffers, use the default Write-Back cacheable setting for best performance

---

## 6. Summary

| Scenario              | Recommended Style          | Cache ON Throughput | Notes                     |
| --------------------- | -------------------------- | ------------------: | ------------------------- |
| Memory Copy           | **HybridV2** (indexed)     |          371 MB/s   | Same as pure WordCopy     |
| Write to Fixed Reg    | **RegWriteV2** (indexed)   |          413 MB/s   | Fastest, exceeds mem copy |
| Read from Fixed Reg   | **RegReadIndexed**         |          373 MB/s   | Indexed is optimal        |

**Key Conclusion**: On Cortex-M55, the indexed counting loop `for (j=0; j<N; j++) arr[j]` is the fastest and most consistent pattern, regardless of cache configuration or whether the operation is memory copy or register I/O.
