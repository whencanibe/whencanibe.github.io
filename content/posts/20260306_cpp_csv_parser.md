+++
title = "How Fast Can You Parse a CSV? A 4-Phase Deep Dive"
date = "2026-03-06"
description = "From ifstream to ARM NEON SIMD — A 4-Phase Deep Dive into C++ CSV Parser Optimization"
tags = ["c++", "performance", "simd", "mmap", "systems-programming"]
categories = ["systems", "cpp"]
+++

# How Fast Can You Parse a CSV? A 4-Phase Deep Dive

I built a CSV parser four different ways to understand how low-level C++ optimizations actually affect performance. The short version: some tricks made things **33x faster**, and others barely moved the needle at all — and understanding *why* was the whole point.

Github link: https://github.com/whencanibe/cpp_csv_parsers

---

## The Setup

Each phase parses the same large CSV file (~500MB, 10 million rows). I measured two things separately:

- **Scan only** — how fast can we find delimiters (commas and newlines) without building any data structures?
- **Full parse** — how fast can we return actual parsed fields the caller can use?

This split turns out to be important.

### Scan Only (delimiter search speed)

| Phase | Time (ms) | Speed | vs. Phase 1 |
|-------|----------:|-------|-------------|
| Phase 1 — ifstream | 1,931.3 | 298.7 MB/s | baseline |
| Phase 2 — string_view | 102.7 | 5,616.4 MB/s | 18.8x faster |
| Phase 3 — mmap | 140.0 | 4,119.9 MB/s | 13.8x faster |
| Phase 4 — NEON SIMD | 57.8 | 9,981.5 MB/s | 33.4x faster |

### Full Parse (returning usable data to the caller)

| Phase | Time (ms) | Speed | vs. Phase 1 |
|-------|----------:|-------|-------------|
| Phase 1 — ifstream | 8,492.6 | 67.9 MB/s | baseline |
| Phase 2 — string_view | 2,555.7 | 225.7 MB/s | 3.3x faster |
| Phase 3 — mmap | 2,484.8 | 232.1 MB/s | 3.4x faster |
| Phase 4 — NEON SIMD | 2,507.7 | 230.0 MB/s | 3.4x faster |

The scan numbers are exciting. The full-parse numbers tell a more humbling story.

---

## Phase 1: The Naive Way (ifstream + getline)

```cpp
std::ifstream file(path);
std::string line;
while (std::getline(file, line)) {
    // split line by commas into std::string fields
}
```

This is the obvious first approach. `ifstream` reads data from disk into a kernel buffer, then copies it into user space. Then `getline` copies each line into a `std::string`. Then we split by commas and copy each field into *another* `std::string`.

That's a lot of copying. Every field allocation hits the heap. For a 10-million-row file, that's tens of millions of `std::string` constructions.

**Scan: 298.7 MB/s. Full parse: 67.9 MB/s.**

---

## Phase 2: Zero-Copy with string_view

`std::string_view` is just a pointer + length — it doesn't own any memory. Instead of copying each field into a new string, we read the entire file into one buffer once, then hand out views into that buffer.

```cpp
// Read whole file into a single std::string (one allocation)
std::string buffer = read_file(path);

// Fields are string_views pointing into buffer — no copies
CsvField field = std::string_view(start, length);
```

The scan jumped from 298.7 → **5,616.4 MB/s** — an 18.8x improvement. We eliminated all the per-field allocations, and the delimiter search loop is tight enough that clang's auto-vectorizer kicks in with NEON instructions automatically (with `-march=native`).

Full parse improved too: 67.9 → **225.7 MB/s** (3.3x). But notice — the gains are much smaller here than in the scan. That's a hint about where the real bottleneck is.

---

## Phase 3: Memory-Mapped Files (mmap)

With `mmap`, the OS maps the file directly into your process's virtual address space. There's no explicit "read file into buffer" step — the page cache *is* your buffer. You access file data through a pointer, and the kernel handles paging it in on demand.

```cpp
int fd = open(path, O_RDONLY);
size_t size = file_size(fd);
const char* data = (const char*)mmap(nullptr, size, PROT_READ, MAP_PRIVATE, fd, 0);
// data is now a pointer directly into the page cache
```

Without mmap: disk → page cache → user buffer (data copied twice in kernel).
With mmap: disk → page cache (which *is* your buffer — zero extra copies).

Full parse: **232.1 MB/s** — slightly faster than phase 2 because we eliminated the initial file-to-buffer copy.

But here's the surprising part: **scan speed dropped** from 5,616 MB/s (string_view) to 4,119 MB/s (mmap). Why? The string_view scan loops over a contiguous `std::string` buffer, which clang auto-vectorizes with NEON. The mmap loop accesses the same memory layout but triggers different compiler heuristics — it stays scalar. Same data, different code path, noticeably slower scan.

---

## Phase 4: ARM NEON SIMD

NEON is ARM's SIMD (Single Instruction, Multiple Data) instruction set, available on every Apple Silicon chip. Instead of checking one byte at a time for commas and newlines, we check 16 bytes simultaneously using vector registers.

```cpp
#include <arm_neon.h>

uint8x16_t chunk     = vld1q_u8(ptr);          // load 16 bytes
uint8x16_t cmp_comma = vceqq_u8(chunk, v_comma); // compare all 16 vs ','
uint8x16_t cmp_nl    = vceqq_u8(chunk, v_nl);    // compare all 16 vs '\n'
uint8x16_t delimiters = vorrq_u8(cmp_comma, cmp_nl); // OR the masks
```

Each `vceqq_u8` sets a byte to `0xFF` wherever there's a match, `0x00` otherwise. We get a 16-byte mask we can scan for hits.

Scan speed: **9,981.5 MB/s** — 33x faster than ifstream, and the first phase to approach memory bandwidth limits on M-series hardware.

Full parse: **230.0 MB/s**. Essentially identical to phases 2 and 3.

---

## The Lesson: Amdahl's Law

Here's what the numbers are really saying:

Phases 2, 3, and 4 all parse at ~225–232 MB/s, even though their delimiter-search speeds vary wildly (5,616 vs 4,119 vs 9,981 MB/s). The scan got 33x faster but the full-parse result barely changed.

This is **Amdahl's Law** in action: *the speedup of a program is limited by the fraction of the program that you actually optimized.*

The NEON scanner eliminates the time spent finding delimiters — but that was never the bottleneck. The bottleneck was (and remains) memory allocation: building the vector of rows and string_views that gets returned to the caller. You can't SIMD your way out of a malloc-heavy data structure.

The real takeaway isn't "SIMD is fast" (though it is). It's: **measure before you optimize, and optimize the thing that actually limits you.** In a real CSV parser, you'd want to profile the caller's usage pattern and possibly avoid returning an owned data structure at all — process rows as a stream instead.

---

## What I Actually Learned

Going through these four phases made abstract concepts concrete in a way that just reading about them doesn't:

- **string_view** isn't just a "more efficient string" — it's a fundamentally different ownership model that changes how you design APIs
- **mmap** removed a memory copy, but lost the auto-vectorization clang gave us for free on the string_view loop
- **SIMD** gives you raw throughput, but only matters if the thing you're speeding up is actually the bottleneck
- The gap between "scan speed" and "parse speed" is a measurement of how expensive your data structures are

Sometimes the most interesting result is the one that *didn't* improve.


