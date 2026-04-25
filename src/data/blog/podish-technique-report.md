---
author: MeoKit
pubDatetime: 2026-04-25T14:00:00Z
title: "Building Podish: An iOS-optimized Linux x86 container (faster than iSH)"
slug: podish-technique-report
featured: true
draft: false
tags:
  - ios
  - tech
  - emulator
description: "A deep dive into Building Podish, a high-performance Linux x86 container specifically optimized for iOS and Apple Silicon."
---

[中文版 (Chinese Version)](/posts/podish-technique-report-zh)

![Podish Icon](../../assets/images/podish-icon.png)

# Building Podish: An iOS-optimized Linux x86 container (that happens to be faster than iSH)

> **TL;DR**: `Podish` is a high-performance Linux x86 user-space container optimized specifically for iOS and Apple Silicon. I wrote an i686 interpreter core in C++ and a Linux compatibility layer in C#. On an iPhone 17 (A19), it scores ~3400 on CoreMark, which is about twice as fast as iSH. 
> 
> Web Demo: [https://podish.meokit.com](https://podish.meokit.com)
> 
> GitHub: [https://github.com/meokit/podish](https://github.com/meokit/podish)

For the past few months, I've been spending my weekends tinkering with a personal project: a cross-platform Linux x86 user-space container heavily optimized for iOS and Apple Silicon.

I’m calling it `Podish`. My goal wasn't to reinvent the wheel or build another UTM, but rather to see how efficiently I could run x86 user-space programs on platforms where JIT compilation is heavily restricted (yes, looking at you, iOS).

![Fastfetch on iPhone](../../assets/images/podish-iphone-fast-fetch.jpeg)

I've been a user of `iSH`, and while the functionality is quite similar, my project is written completely from scratch without using any of their code. To my pleasant surprise, it currently outperforms `iSH` in quite a few areas.

If you’d like to try it in your browser, you can visit the [Podish Web Demo](https://podish.meokit.com). Please note that the web version is currently slower and lacks networking support.

![Browser Demo](../../assets/images/podish-browser.png)

A huge source of inspiration for the interpreter was [LuaJIT Remake](https://github.com/luajit-remake/luajit-remake). I learned a tremendous amount from studying that codebase, and I'm very grateful to its creator.

By day, I'm a game programmer who works with various scripting languages. I'm well aware of the infamous iOS limitation: no JIT compilation allowed. Specifically, you can't map memory pages as WX (Write-Execute), meaning you can't execute unsigned code. For instance, you can't run LuaJIT in JIT mode on iOS; you're stuck with the performance-limited `-joff` mode. Similarly, you can't get the fast, JIT-accelerated UTM on the App Store—only the painfully slow UTM SE.

This got me thinking: *Could a pure interpreter ever rival JIT? Just how fast can an interpreter actually get?*

I know there are masterpieces out there like LuaJIT's `-joff` mode and Wasm3, which are blazing fast, but I really wanted to try writing one myself. After looking at the source code for iSH—which contains a massive amount of hand-written assembly—I wanted my emulator to execute x86 code efficiently, but I honestly didn't want to write assembly by hand.

Through LuaJIT Remake, I learned about Clang's "new" ABI features: `preserve_none`, `preserve_all`, and `[[musttail]]`. For an interpreter, dispatch overhead is often the biggest bottleneck. Could `preserve_none` + `[[musttail]]` generate code comparable to hand-written assembly? I had no idea, but it sounded like a fun experiment.

Around that time, I had just subscribed to Gemini Advanced. I used it to help me write the mountain of boilerplate code required for the interpreter—like extracting test cases from the massive Intel manuals and drafting the actual instruction handlers. I hand-wrote the instruction dispatch loop and the core data structures, disassembled the i686 build of Redis, had Gemini write a script to filter out unique instruction patterns, and then generated test cases for each one.

Week 1: Success! I got a simple `hello world` running. It was a very basic program that used `int 3` to invoke the `write` syscall to stdout. My emulator intercepted the instruction and printed the output.

After that, I started writing the Linux syscall compatibility layer in C#. About a month later, I finally got `CoreMark` to run. 

In the early stages, it only scored about `600` in `CoreMark`. After several rounds of highly specific optimizations, it stabilized at `~3000` on my 2023 M3 Max MacBook Pro, and impressively, hit `~3400` on an iPhone 17.

![CoreMark on iPhone](../../assets/images/podish-iphone-coremark.jpeg)

Today, it can run Busybox, Bash, Python, LuaJIT, GCC, OpenSSH, and even Node.js. I even managed to boot up the Gemini CLI, though it crashes fairly quickly. Turning off V8's JIT makes it slightly more stable, but performance drops to an unusable level.

![Gemini CLI on iPhone](../../assets/images/podish-iphone-gemini-cli.jpeg)
![Gemini CLI on macOS](../../assets/images/podish-gemini-cli-macos.png)

Here is a rough outline of the current repository layers:

- `libfibercpu`: IA-32 emulator core written in C++
- `Fiberish.Core`: Linux runtime / kernel compatibility layer
- `Fiberish.Netstack`: Based on [smoltcp](https://github.com/smoltcp-rs/smoltcp)
- `Podish.Core`: Higher-level container/runtime orchestration
- `Podish.Cli`: CLI for actual usage
- SwiftUI / Browser interfaces

I chose C# for the syscall emulation because its cross-platform abstraction layer is excellent, which fit my portability needs perfectly. Plus, coming from a Unity game development background, C# is my comfort zone.

```text
Guest i686 Linux Binary
  -> Predecode / Block Builder
  -> Interpreter Dispatch + Semantics
  -> MMU / MicroTLB
  -> Linux Runtime / Syscall Layer
  -> VFS / Process / Signal / Network
  -> Host OS / Browser
```

## Crossing the Language Boundary: C++ Core and C# Runtime

A lot of readers might immediately wonder: *Why use C++ for the core but C# for the syscalls? Doesn't the boundary overhead eat up all your performance gains?*

My reasoning was twofold:

**1. Why C#?** Early on, I needed to rapidly implement the semantics for about 200 Linux syscalls, a VFS layer, a network stack, and container lifecycle management. C#'s cross-platform I/O, string handling, async model, and rich standard library allowed me to push from "hello world" to a running `CoreMark` in just a month. If I had used pure C++, I’d probably still be writing wrappers for `std::filesystem`.

**2. How bad is the boundary overhead?** To prevent P/Invoke from becoming a bottleneck, I made a few key design choices:

* **Minimal C API**: `libfibercpu` exposes a strictly C-style API (`X86_Create`, `X86_Run`, `X86_RegRead`, etc.), which C# calls via `LibraryImport` / `DllImport`. The `EmuState` is just an `IntPtr` on the C# side, creating zero GC pressure.
* **Zero-copy memory access**: The C# layer never reads/writes guest memory byte-by-byte through P/Invoke. Instead, `bindings.h` provides `X86_ResolvePtrForRead/Write` returning a raw host pointer, which C# maps using `Span<byte>` to manipulate directly.
* **Anchoring Callbacks**: Callbacks into C# (Fault, Interrupt, Log) need to hold a reference to the C# `Engine` object. I pin the object on the heap using `GCHandle.Alloc(this)` and pass the pointer to C++ as `userdata`, completely avoiding GC relocation issues across the boundary.
* **Batching Syscalls**: In the hot path, the guest program usually executes hundreds or thousands of instructions in C++ before triggering a single syscall. The actual frequency of boundary crossings depends on the guest's syscall density, not its instruction density.

Profiling shows that in compute-heavy workloads (like CoreMark), language boundary overhead accounts for less than 1%. In I/O-heavy workloads (like `git clone`), the bottlenecks are the network and VFS, not P/Invoke.

## SMC and JIT Guests

I mentioned earlier that getting LuaJIT to run forced me to implement executable-page writes and block cache invalidation correctly. Dealing with SMC (Self-Modifying Code) is a classic emulator headache. Modern JIT engines (V8, LuaJIT, .NET) write machine code to memory and immediately jump to it.

Instead of trying to monitor every write operation globally, my approach **reuses the MMU's permission system**:

* When a host memory page is mapped as both **Executable** and **Writable** (typically via `mmap` with `PROT_EXEC|PROT_WRITE`), the MMU internally marks it as an `External Alias`.
* If `BasicBlocks` have already been pre-decoded and cached on this page, the MMU tags the write permission for this page with a `ForceWriteSlow` flag.
* Any subsequent write to this page will miss the fast TLB, hit `ForceWriteSlow`, and be forced into a slow path. The slow path calls `invalidate_code_cache_page(addr)`, invalidating all `BasicBlocks` associated with that page.
* Most importantly: If the interpreter detects that the page containing the current EIP is being modified (`ShouldInterceptExecWriteForSmc`), it immediately yields and switches to a single-instruction "safe mode" to prevent a race condition between "old code currently running" and "new code just written."

This mechanism allows LuaJIT's `-joff` mode to run stably and even lets Node.js/V8 start up. However, likely due to subtle bugs in my instruction set implementation or incomplete Linux syscalls, it still crashes occasionally—a limitation I'm actively looking into.

### SMC Mechanics in Detail

The overview above covers the high-level idea, but if you're building something similar, the details below are worth reading. The core philosophy is to **push all detection costs into the MMU permission bits**, avoiding global write monitoring or disassembly scans.

**External Alias Tracking.** `MmuCore` maintains an `external_aliases` map internally (key is the host page pointer; value contains `exec_count`, `write_count`, and the set of associated guest pages). When the C# layer maps a host page as `External + Write + Exec` via `mmap`, that page gets registered in this table. This table is only updated on page-table changes (`mmap`/`mprotect`/`munmap`); the hot path never touches it.

**Lazy Arming of ForceWriteSlow.** What actually determines whether an external page needs SMC detection is not whether it has `Exec` permission, but **whether there are cached BasicBlocks on it**. `refresh_smc_armed_for_host_page()` is called in two situations:

1. A new block has been decoded, and we need to check if any of the host pages it covers have writable external mappings;
2. An external mapping is established or modified (e.g., `mprotect` adds `PROT_WRITE`).

If `(exec_count > 0) && (there is a live block on this host page)`, all guest page entries for this page are tagged with `ForceWriteSlow`. This bit is a **runtime temporary mark**; it is not persisted, not copied by `fork`, and does not appear in deep copies of the page table.

**Write-path Interception Chain.** When the guest executes a write instruction:

```text
write() -> MicroTLB miss -> SoftTLB miss -> resolve_slow()
  -> look up page directory -> find ForceWriteSlow in permissions
  -> call invalidate_code_cache_page(guest_addr)
  -> mark all BasicBlocks on the associated host page as invalid
  -> continue with the actual memory write
```

`invalidate_code_cache_page` does not traverse the entire block cache. Instead, it uses a `page_to_blocks` reverse index (host page → block list) to achieve O(number of affected blocks) local invalidation. Blocks marked invalid are automatically re-decoded the next time `X86_Run` tries to enter them.

**Execution-time Race: Old Code Currently Running vs. New Code Just Written.** Merely invalidating the block cache on write is not enough—if the current EIP happens to fall on the page being written, the tail-call jump to the next instruction may already have been overwritten. For this, `mmu_impl.h` has `ShouldInterceptExecWriteForSmc`:

```cpp
if (state->intercept_exec_write_for_smc && !state->allow_write_exec_page) {
    uint32_t current_page = state->ctx.eip >> 12;
    uint32_t target_page  = addr >> 12;
    if (target_page == current_page || target_page == current_page + 1)
        return true;  // trigger SMC yield
}
```

Once this condition hits, the write does not execute immediately. Instead, `state->smc_write_to_exec` is set, and the current handler returns a special yield flow. When `X86_Run`'s main loop detects `smc_write_to_exec`, it enters single-instruction safe mode:

- Set `allow_write_exec_page = true`, allowing **exactly one** guest instruction to write to an executable page;
- Decode and execute only a single-instruction block for the current EIP (`max_insts = 1`);
- Immediately clear `allow_write_exec_page` after execution, restoring the interception state.

This guarantees a clear instruction boundary between the write operation and the jump to new code, preventing races.

**TLB Consistency Under Multi-Engine Sharing.** When `clone(CLONE_VM)` creates a new thread, multiple `Engine`s share the same `MmuCore`, but each `Engine` has its own `SoftTLB` and `block_lookup_cache`. Engine A's `invalidate_code_cache_page` does not automatically flush Engine B's local TLB. For this, `MmuCore` has a `RuntimeTlbShootdownRing` (1024-slot ring buffer):

- The party that changed the page table writes the flushed guest page into the ring;
- Other Engines call `sync_runtime_tlb_shootdowns()` the next time they enter `X86_Run`, consuming new entries in the ring and flushing their local TLBs.

This ring's capacity is large enough to cover the vast majority of single `mprotect`/`munmap` page ranges; if it overflows, a full flush is issued directly.

Here is a rough, non-exhaustive list of guest apps I've tested so far:

| Category | Example | Current Status | Notes |
| :--- | :--- | :--- | :--- |
| Shell / Base Userland | `busybox` / `ash` | Stable | Essential for CLI interaction; ran `vim` successfully. |
| Complex Shell | `bash` | Mostly usable | The shell itself works, but complex toolchains still face issues. |
| Scripting Runtime | `python3` | Stable (Benchmarks) | Haven't tested massive Python projects yet. |
| JIT / SMC | `LuaJIT` | Stable (Benchmarks) | Forced me to get the block-cache invalidation right. |
| Build Tools | `gcc` / `make` | Verified | Can run `make compile` on CoreMark; haven't tried compiling huge codebases. |
| Network / Dev Tools | `git` / `OpenSSH` | Works manually | Exposed bugs in VFS, PTY, and network boundaries. |
| Heavy Modern Runtime | `Node.js` | Boots, but unstable | Can run npm occasionally, crashes often. Slower but stabler with V8 JIT off. |
| AI CLI | `Gemini CLI` | Boots, then crashes | Blocked by the Node/V8 instability. |

![Vim on iPhone](../../assets/images/podish-iphone-vim.jpeg)
![Yazi on iPhone](../../assets/images/podish-iphone-yazi.jpeg)

## A Failed Experiment with Copy-and-Patch JIT

The concept of a Copy-and-Patch JIT is beautifully simple: pre-compile opcode handlers into binary stencils, and at runtime, just copy the template and patch in the constants/addresses. It bypasses the complexity of register allocation and heavy code generation. 

Since I was already using Clang's `preserve_none` and `[[musttail]]` to make my handlers resemble hand-written assembly, turning them into stencils seemed like a logical next step. I already had the handlers; I just needed to patch the operand fetching and string them together.

I was hoping for a 200%+ performance boost. The result? **It was actually slightly slower than my interpreter.**

In hindsight, this makes sense. Copy-and-patch saves you the cost of the interpreter loop continuously decoding and dispatching logic. However, my direct-threaded interpreter had already pushed dispatch overhead incredibly low. My real bottlenecks were memory access, address translation, state maintenance, and I-Cache pressure. By generating stencils, I eliminated some bytecode reading, but the resulting code bloat absolutely wrecked the I-Cache. 

It was a humbling lesson: a poorly fitted JIT is just garbage, and loading immediates isn't necessarily faster than reading bytecode if your cache is thrashing.

This forced me to profile more carefully:

- Once a direct-threaded interpreter has pushed dispatch cost low, the bottleneck may no longer be dispatch
- For this design, memory access and address translation are more expensive than I expected
- A bad JIT is garbage: I-Cache pressure increases, and loading immediates is not necessarily faster than loading bytecode

## Core Idea 1: Pre-decoding + Tail-call Dispatch

The architecture I settled on:
* Pre-decode x86 instructions into a fixed-length Intermediate Representation (IR).
* Store them sequentially in a Basic Block.
* Attach the next guest PC, operand descriptions, and execution entry point to every IR.
* Instead of returning to a central dispatcher, directly `tail-call` into the next instruction's handler.

This shifts the burden of decoding variable-length x86 instructions upfront and caches the result, significantly reducing loop branches. To my absolute surprise, this approach turned out to be roughly 10x faster than QEMU TCI. (A big shoutout to Justine Tunney’s [blink](https://github.com/jart/blink) project, which introduced me to Intel's [XED](https://github.com/intelxed/xed) and helped me build the table-driven decoder).

A pre-decoded intermediate representation looks roughly like this:

`DecodedOp` is fixed at **32 bytes**, aligned to a 16-byte boundary:

| Field | Type | Offset | Size | Description |
| :--- | :--- | ---: | ---: | :--- |
| `handler` | `HandlerFunc*` (function pointer) | 0 | 8 | Entry point for the instruction's semantics |
| `next_eip` | `uint32_t` | 8 | 4 | PC of the next instruction |
| `len` | `uint8_t` | 12 | 1 | Instruction byte length |
| `modrm` | `uint8_t` | 13 | 1 | ModRM byte |
| `prefixes` | `Prefixes` (union) | 14 | 1 | Prefix info (lock/rep/segment…) |
| `meta` | `Meta` (union) | 15 | 1 | Meta info (has_mem / has_imm / ext_kind…) |
| `ext.data` | `DecodedMemData` | 16 | 16 | Memory operand description (imm / ea_desc / disp) |
| `ext.link.next_block` | `BasicBlock*` | 24 | 8 | Cached successor block pointer after block linking |
| `ext.control` | `DecodedControlFlowData` | 16 | 16 | Control-flow target (target_eip / cached_target) |

If we take a simple memory ALU instruction as an example:

```text
Guest:
  add eax, [ebx+4]

Predecoded IR:
  entry      = op_add_r32_rm32
  next_pc    = 0x...
  operands   = { dst=eax, src=mem(base=ebx, disp=4, size=4) }
  flags_mode = arithmetic
  control    = fallthrough
```

## Core Idea 2: Parity-Only Lazy Flags

Implementing x86 `EFLAGS` accurately is tedious. QEMU uses a very elegant [lazy evaluation system](https://qemu.weilnetz.de/w64/2012/2012-12-04/qemu-tech.html) where they store `CC_SRC`, `CC_DST`, and `CC_OP`, only calculating the flags when explicitly requested. 

I tried implementing this, and it was *slower* than calculating them eagerly. Why? Because writing those three variables to memory and reading them back was killing me—I was already memory-bound.

I compromised: I only evaluate the Parity Flag (PF) lazily. CF/ZF/SF/OF are written/read by almost every ALU and jump instruction anyway, so doing that lazily was adding memory overhead. PF, however, is expensive to calculate (requires iterating bits) and rarely used (only by `JP`/`JNP`). 

I carry a `uint64_t flags_cache` in a register through the tail-call chain. The lower 32 bits are real-time EFLAGS, and the upper bits hold the raw byte for lazy PF calculation. It’s an imperfect, highly-specific middle ground, but it works wonderfully for this architecture.

**Static layer:** Within a basic block, I do def-use analysis on flags. If an instruction writes flags but no one reads them later, it is replaced directly with a no-flags handler variant. This variant is expressed through template parameters like `AluAdd<T, false>` and `AluSub<T, false>`; the compiler completely elides the entire flags-update path.

**Dynamic layer:** During execution of each instruction, `flags_cache` is passed as a register argument through the tail-call chain and is never written back to memory. Only `PF` is lazy:

- When writing PF, `SetParityState(flags_cache, result_byte)` stuffs the low 8 bits of the result directly into the high parity state bits without calculating parity;
- When reading PF, `EvaluatePF(flags_cache)` retrieves the source byte from the high bits and calculates parity on the spot;
- At externally visible points (`pushf`, `lahf`, faults, interrupts, API boundaries), `GetArchitecturalEflags()` materializes the lazy parity and returns the full 32-bit EFLAGS.

**Commit semantics:** `CommitFlagsCache` is only called once at handler chain exit boundaries (`ExitOnCurrentEIP`, `ExitOnNextEIP`, restart/retry, resolver miss), writing the register-held `flags_cache` back to `state->ctx.flags_state`. During successful chaining it is never committed—because the next instruction's handler will continue passing the same `flags_cache` as a register argument. `X86_Run()` and `X86_Step()` also do not commit after the handler returns, to avoid overwriting updated state with a stale caller-side copy.

An interesting detail is the `CheckCondition` LUT path: the vast majority of conditional jumps (cond 0-9, 12-15) do not depend on PF, so they read the low 32 bits via `GetFlags32Raw(flags_cache)` and use a LUT lookup to decide the branch direction; only cond 10/11 (JP/JNP) branch separately to `EvaluatePF()`. This design makes the extra cost of PF laziness nearly zero.

## Core Idea 3: Memory Access is the Real Bottleneck

I spent so much time analyzing dispatch overhead, only to realize during profiling that my biggest issue was shockingly basic: my TLB was constantly missing, causing massive Page Table Walk overhead.

It turned out to be a very silly TLB refill bug. Fixing it immediately bumped my CoreMark from 600 to 800. It made me realize: **Address translation is the true bottleneck.**

I introduced a `MicroTLB`—a tiny 16-byte structure kept permanently in a host register during the execution chain. If it hits, it's just a quick tag match and an addition (`host_ptr = guest_addr + addend`). Even with a 50% hit rate, avoiding the memory read to the main `SoftTLB` gave a massive performance boost.

`SoftTLB` is a three-way direct-mapped TLB, consisting of three fixed-size tables:

| Field | Type | Offset | Size | Description |
| :--- | :--- | ---: | ---: | :--- |
| `read_tlb` | `std::array<TlbEntry, 256>` | 0 | 4096 | Fast read-permission mapping table |
| `write_tlb` | `std::array<TlbEntry, 256>` | 4096 | 4096 | Fast write-permission mapping table |
| `exec_tlb` | `std::array<TlbEntry, 256>` | 8192 | 4096 | Fast execute-permission mapping table |

`TlbEntry` is aligned to 16 bytes, fixed at **16 bytes**:

| Field | Type | Offset | Size | Description |
| :--- | :--- | ---: | ---: | :--- |
| `tag` | `uint32_t` | 0 | 4 | Guest page tag (high 20 bits) |
| `perm` | `Property` (enum) | 4 | 4 | Permission bits (Read / Write / Exec / Dirty …) |
| `addend` | `std::uintptr_t` | 8 | 8 | `host_ptr = guest_addr + addend` |

Indexing is also very direct:

```text
idx = (guest_addr >> PAGE_SHIFT) & 255
tag = guest_addr & ~PAGE_MASK
host_ptr = guest_addr + addend
```

In other words, `SoftTLB`'s job is to compress already-looked-up guest page mappings into a fast `tag + addend` entry. On a hit, you just check the tag and do one addition to get the host address; on a miss, you fall back to the slow path to check permissions, fill the entry, and handle exceptions.

`MicroTLB` is a resident register structure, aligned to 16 bytes, fixed at **16 bytes**:

| Field | Type | Offset | Size | Description |
| :--- | :--- | ---: | ---: | :--- |
| `tag_r` | `uint32_t` | 0 | 4 | Read-permission tag, default `0xFFFFFFFF` (miss state) |
| `tag_w` | `uint32_t` | 4 | 4 | Write-permission tag, default `0xFFFFFFFF` (miss state) |
| `addend` | `std::uintptr_t` | 8 | 8 | `host_ptr = guest_addr + addend` |

Note the data layout: `read_tag` + `write_tag` are merged into the same register, so this structure occupies two host registers.

Its hit/miss path can be represented as:

```text
guest address
  -> check read_tag / write_tag
  -> hit: host_ptr = guest_addr + host_guest_addend
  -> miss: full translation + permission check + refill MicroTLB
```

During refill, permissions are checked; if there is no read permission, `read_tag` is cleared, and vice versa.
This design may seem odd—if reads and writes repeatedly hit different pages, the MicroTLB will ping-pong and the hit rate drops to zero.
But most of the time it works. My measurements show a hit rate above 50%. As long as there is some hit rate, reducing the frequency of reads from the in-memory SoftTLB provides a performance boost.

## Core Idea 4: Block Linking and Superopcodes

My `BasicBlock` isn't just an array of instructions; it contains branching targets, fall-through addresses, and execution counters.

If a block is short, doesn't cross page boundaries, and has simple control flow, I stitch the next block directly to the end of it (**Block Linking**), saving indirect memory jumps.

For **Superopcodes**, a naive bigram frequency count didn't help much. Instead, I profiled the execution, found hot "anchor" instructions, and fused them with adjacent instructions *only* if there was a Read-After-Write (RAW) dependency (e.g., `test eax, eax ; je ...` or `pop ebx ; pop esi`). Fusing about 256 of these patterns pushed the CoreMark past the 3000 barrier.

In this interpreter, a `BasicBlock` is an object with a fixed-size header followed by a variable-length instruction stream. Its header looks roughly like this:

`BasicBlock` is aligned to 16 bytes; the header is fixed at **48 bytes**, followed by a variable-length `DecodedOp` array:

| Field | Type | Offset | Size | Description |
| :--- | :--- | ---: | ---: | :--- |
| `chain.start_eip` | `uint32_t` | 0 | 4 (embedded in `BasicBlockChainPrefix`) | Block start guest PC |
| `chain.inst_count` | `uint8_t` | 4 | 1 | Number of instructions in the block |
| `chain.valid` | `uint1_t` | 5 | 1 bit | Whether the block is valid (for invalidation marking) |
| `entry` | `HandlerFunc*` | 8 | 8 | Semantic entry point of the first instruction; the interpreter jumps here directly |
| `end_eip` | `uint32_t` | 16 | 4 | Block end address (next_eip of the last instruction) |
| `slot_count` | `uint32_t` | 20 | 4 | Total slot count (including trailing sentinel) |
| `sentinel_slot_index` | `uint32_t` | 24 | 4 | Index of the sentinel in the slots array |
| `branch_target_eip` | `uint32_t` | 28 | 4 | Branch target address (valid for conditional/direct jumps) |
| `fallthrough_eip` | `uint32_t` | 32 | 4 | Fallthrough address |
| `terminal_kind_raw` | `uint8_t` | 36 | 1 | Terminal kind (None / DirectJmpRel / DirectJccRel / Other) |
| `block_padding0` | `uint8_t` | 37 | 1 | Padding |
| `block_padding1` | `uint16_t` | 38 | 2 | Padding |
| `exec_count` | `uint64_t` | 40 | 8 | Number of times the block has been executed (for profile-guided superopcode) |
| `slots[]` | `DecodedOp[]` | 48 | Variable (32 bytes each) | Pre-decoded instruction stream |

A few fields are especially critical:

- `entry` points to the first executable semantic entry of this block; once the interpreter has the block, it can start running from here directly.
- `slots[]` is a sequentially ordered `DecodedOp` array, each `DecodedOp` fixed at `32` bytes.
- `slot_count` includes the trailing sentinel, so the block's memory layout is a fixed header plus a contiguous op stream.
- `branch_target_eip` / `fallthrough_eip` let block linking know successor destinations at the block level ahead of time.
- `exec_count` is important input for profile-guided superopcode generation.

`DecodedOp` itself is not a minimal "handler-pointer-only" structure. Beyond `handler` and `next_eip`, it also stores memory operand info, control-flow targets, or the `next_block` pointer cached after block linking in its extension area.

So what `BasicBlock` really does is bind three things together:

- Caching pre-decode results to avoid repeated decoding
- Placing a sequence of `DecodedOp`s compactly into a tail-callable execution stream
- Providing a stable carrier for subsequent optimizations like block linking, superopcode, and profiling

When I later implemented block linking, I stitched and reused directly on top of the existing `BasicBlock`.

### Block Linking

If a basic block is short enough, its instructions don't cross page boundaries, and its control flow is simple, the successor block is stitched directly to the end of the current block. This reduces some indirect memory access during inter-block jumps.

### Superopcode

At first I tried simple bigram statistics, but the results were mediocre. The reason is obvious: high-frequency bigrams aren't necessarily frequently executed—they just have high frequency among all pairs, and the total number of fixed instruction pairs isn't large to begin with. Additionally, because we do instruction specialization, the specialized instructions become extremely sparse, making even fewer pairs fixable.

The more effective approach is:

- First select seeds around high-frequency anchor instructions
- Then examine their def-use relationship with neighboring instructions
- Only fuse combinations with `RAW` (Read-After-Write) dependencies

This strategy ultimately generated about `256` superopcodes, and was one of the key steps in stably pushing performance past `3000+ CoreMark`.

Representative examples in the current generation set include:

- Stack-operation chains: `pop ebx ; pop esi`
- Flags producer/consumer chains: `test eax, eax ; je/jz ...`
- Load-use chains: `mov eax, [esp+off] ; sub reg, eax`
- Load-store chains: `mov ebx, [esp+off] ; mov [esi], ebx`

The candidate discovery pipeline is essentially not complicated:

```text
block trace
  -> hotspot statistics
  -> anchor selection
  -> def-use / RAW filtering
  -> code generation
  -> regression verification and benefit review
```

## The Journey from 600 to 3000

Optimizing this was a lesson in how quickly bottlenecks shift. Text-book solutions don't always fit specific constraints. 

| Phase | Key Change | CoreMark | Notes |
| :--- | :--- | :--- | :--- |
| Initial | Baseline interpreter | ~600 | Logic was there, but a TLB bug ruined the hot path. |
| Bugfix | Fixed address translation | ~800 | Confirmed address translation was the biggest issue. |
| Hot Path | PC/block budget tuning | ~1500 | Stopped writing redundant state to memory. |
| Memory | Data layout / paired loads | ~2000 | Shifted focus from dispatch to memory access. |
| Lazy Flags| Static pruning + PF lazy | ~2200 | Found the sweet spot for flag evaluation. |
| Linking | Append simple blocks | ~2500 | Lowered block boundary costs. |
| Superops | Profile-guided fusion | ~3000 | Fused ~256 hot instruction pairs. |

## Two "Counter-Intuitive" Data Points

### 1. Why is the A19 faster than the M3 Max?
In this specific project, the interpreter is entirely bound by single-thread IPC and memory latency. It can't leverage the M3 Max's massive multi-core architecture or 300GB/s bandwidth. CoreMark fits easily in the cache, so the A19's slight lead is just a reflection of architectural generation differences in single-core latency.

### 2. `luajit -joff` is faster than the JIT on iSH
When running `primes_jit.lua` on iSH, the pure interpreter mode (`-joff`) took 27s, while the JIT mode took 46s. 
This defies the "JIT is always faster" rule. My theory is that iSH's SMC detection is very heavy. LuaJIT's constant code generation causes iSH to continually flush and re-translate its code cache. In `-joff` mode, no machine code is written, SMC traps are avoided, and it runs smoother. *(If anyone knows the iSH codebase well, I'd love to hear your thoughts on this!)*

## Threading Model: Single Scheduler Thread + Cooperative Task Switching

I implemented `clone`/`fork`/`vfork` and basic `pthread` semantics, but **the concurrency model is not OS-level multi-threaded parallelism**.

The architecture is as follows:

- `KernelScheduler` is bound to a fixed host thread (the scheduler thread). All `FiberTask` creation, switching, syscall dispatch, and signal delivery happen sequentially on this thread.
- Each `FiberTask` owns its own `Engine` instance (an independent `EmuState`), so the interpreter core itself **does not need** thread-safety design—there is no situation where two threads read/write the same `EmuState` simultaneously.
- When guest code calls `clone(CLONE_VM | CLONE_THREAD)` to create a new thread, the C# layer creates a new `FiberTask` and a new `Engine`, but both `Engine`s share the same `MmuCore` (lifetime managed by atomic reference counting).
- Sharing `MmuCore` introduces a problem: when Engine A modifies the page table or flushes the block cache, Engine B's `SoftTLB` and `block_lookup_cache` are still stale. I implemented a lightweight `RuntimeTlbShootdownRing` for this: the party that changed the page table writes the affected guest page into a ring buffer, and other Engines consume from this ring and flush their local TLBs before their next execution.

The ceiling of this design is that multiple guest threads are **serialized at the host level** and cannot leverage multi-core parallelism. For shells, Python, and compilation tasks inside the container, this is usually enough; but if the guest runs heavily parallel scientific computing, performance will be noticeably limited.

## What Can It Run Now

Beyond the CPU core, the project also implements a Linux compatibility environment, including:

- Linux-like `fork` / `vfork` / `clone` / `execve` / `wait*` semantics
- `tmpfs` / `procfs` / host mounts / overlay roots
- PTY / TTY
- Sockets and native netstack integration
- OCI image pull/save/load/import/export
- Container-style execution via `Podish.Cli`

This layer is what determines whether the project can move from benchmarks to real-world workloads.

| Category | Example / Evidence | Current Status | Main Limitation | Notes |
| :--- | :--- | :--- | :--- | :--- |
| Shell | `/bin/sh -lc true` | Verified | Currently covers shell startup and short commands | |
| Coreutils / Archive | `grep -R` / `tar cf` / `tar xf` | Verified | | |
| Scripting | `python` (`primes.py`) | Verified | Workload currently has only a few samples; easily affected by phone temperature | |
| JIT-heavy | `LuaJIT` (`primes_jit.lua`) | Verified | Results were once polluted by a notify-socket / network-path bug, now fixed | JIT-related guests work |
| Build-oriented mixed workload | `make compile` on CoreMark tree | Verified | Current main table uses a compile-only metric; each round does `make clean` first, then times literal `make compile` | |
| Network / tooling compat | `git clone` | Compatibility verified | High network variance | Mainly validates VFS / DNS / TLS / git pipeline |

## Current Benchmarks

*(Note: Podish/QEMU desktop data from M3 Max. iSH data from iPhone 17 (A19). Podish/QEMU guests run Alpine Linux i386. Results are medians of 5 runs unless otherwise noted.)*

### Startup & Compute Intensive

| Workload | Podish(A19) | Podish(M3) | iSH (A19) | Podman i386 | QEMU TCI | Native (M3) | 
| :--- | ---: | ---: | ---: | ---: | ---: | :--- |
| `sh -lc true` | 20 ms | 20 ms | 30 ms | 30 ms | 30 ms | 10 ms |
| CoreMark 1.0 | 3447 | 2967 | 1692 | 11456 | 325 | 38087 |
| `python primes.py`| 78.3 s | 89.4 s | 684.4 s | 40.9 s | 787.3 s | 1.8 s | 
| `luajit primes` | 3.1 s | 4.0 s | 46.6 s | 1.7 s | 39.3 s | 0.2 s |
| `luajit -joff` | 14.5 s | 17.3 s | 27.5 s | 7.1 s | 152.7 s | 0.7 s |

*CoreMark looks okay on paper, but complex scripts still suffer heavily, likely due to address translation overhead.*

I also ran `wasm3 coremark.wasm` five times on the same MacBook as another interpreter reference:

- 1st run: `Iterations = 60000`, `CoreMark = 3308.108090`
- Next 4 runs stabilized at `Iterations = 40000`, scores: `3292.879486`, `3246.276318`, `3212.163048`, `3208.350438`
- Median ≈ `3229.22`

In other words, Podish's pure compute performance is already competitive with Wasm3.

### File I/O & Mixed Workloads

| Workload | Podish(A19) | Podish(M3) | iSH (A19) | Podman i386 | Native (M3) |
| :--- | ---: | ---: | ---: | ---: | :--- |
| `grep -R` | 50 ms | 50 ms | 70 ms | 70 ms | 10 ms |
| `tar cf` | 40 ms | 30 ms | 40 ms | 97 ms | 10 ms |
| `tar xf` | 250 ms | 250 ms | 170 ms | 161 ms | 30 ms |
| `make compile` | 9.4 s | 10.4 s | 11.4 s | 7.3 s | 0.2 s |
| `git clone` | 3.2 s | 3.3 s | 5.1 s | 2.6 s | 2.9 s |

*Network/IO benchmarks are mostly for compatibility verification rather than strict performance metrics.*

## A Note on Graphics and Audio

While everything above focuses on CLI and compute, there are actually two very experimental multimedia pipelines hidden in the codebase: `Podish.Wayland` and `Podish.Pulse`. They attempt to bridge guest Wayland/PulseAudio protocols to the host's SDL and Audio backends. Right now, they are rough, unpolished, and macOS only (they don't work on iOS/Web yet), but it’s a fun space I’m exploring.

- **`Podish.Wayland`** (~7K lines of C#): Implements a lightweight Wayland compositor core that bridges guest Wayland client protocols to host SDL windows. The CLI contains `WaylandSdlDisplayHost` and `WaylandSdlFramePresenter`, meaning you can theoretically open a window on the macOS desktop and run Linux GUI programs.
- **`Podish.Pulse`** (~6K lines of C#): Implements PulseAudio protocol client/server sides, redirecting guest audio streams to the host audio backend. The CLI also has `PodishPulseVirtualDaemon` integrated.

Right now, both pipelines are "code exists, architecture is clear, but lacks end-to-end polish." The graphics path currently only supports sending Wayland surfaces via shmem; it does not support gbm/EGL. I tested `foot` and it works, but SDL2 does not—apparently SDL2 on Wayland requires EGL. The audio path is still rough, but `ffplay -nodisp` can play music.

These two features are currently macOS only; iOS / Web adaptations have not been done yet.

![Wayland Foot Terminal](../../assets/images/podish-wayland-foot.png)

***

Building `Podish` has been a fantastic learning experience, showing me just how far you can push a simple direct-threaded interpreter. I'm still figuring a lot of this out, so if you spot any glaring errors in my logic or have ideas for optimization, I’d be thrilled to hear them. 

You can check out the source code here: [GitHub - meokit/podish](https://github.com/meokit/podish)