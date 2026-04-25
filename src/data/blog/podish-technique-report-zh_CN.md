---
author: MeoKit
pubDatetime: 2026-04-25T14:00:00Z
title: "Podish 技术报告：一个比 iSH 更快的 iOS Linux x86 容器"
slug: podish-technique-report-zh
featured: true
draft: false
tags:
  - ios
  - tech
  - emulator
description: "深入分析 Podish 的实现原理，这是一个专为 iOS 和 Apple Silicon 优化的高性能 Linux x86 容器。"
---

[English Version (英文版)](/posts/podish-technique-report)

![Podish 图标](../../assets/images/podish-icon.png)

# 一个比 iSH 更快的 iOS Linux x86 容器

> **TL;DR**：`Podish` 是一个面向 iOS / Apple Silicon 专门优化的高性能 Linux x86 用户态容器。它用 C++ 写了一个 i686 解释器核心，用 C# 写了 Linux 兼容层，在 iPhone 17 (A19) 上跑出 CoreMark ~3400，比 iSH 快一倍。
> Web Demo: https://podish.meokit.com
> GitHub: https://github.com/meokit/podish

我最近几个月的周末一直在做一个项目：跨平台的 Linux x86 用户态容器，并面向 iOS/Apple Silicon 专门优化。

这个项目现在叫 `Podish`。我的目标不是复刻一个 UTM ，而是尽可能高效地在JIT受限的平台（就是你，iOS）上运行 x86 用户态程序。

![iPhone 上的 Fastfetch](../../assets/images/podish-iphone-fast-fetch.jpeg)

我也用过 `iSH`, 这个功能有点类似 `iSH`，但是代码是我从头写的，没有来自 `iSH` 的代码，并且现在不少方面的性能也比 `iSH` 快一截。

如果你想在浏览器中体验，可以直接访问 [Podish](https://podish.meokit.com)，不过网页上暂时性能较差并且没有实现网络功能。

![浏览器演示](../../assets/images/podish-browser.png)

解释器的主要灵感来自于 [LuaJIT Remake](https://github.com/luajit-remake/luajit-remake)
我从 LuaJIT Remake 学到了非常多东西，非常感谢这个项目。

我是一个游戏程序员，接触各种脚本语言。了解到iOS上非常出名的限制：不允许JIT。

实际上是禁止 WX 的内存页映射，禁止执行一切未经签名的代码。比如说，你无法在iOS上运行LuaJIT的JIT模式，只能跑性能受限的 `-joff` 模式。
比如你无法在 App Store 下载带有JIT加速的 UTM，只能下载到极其缓慢的 UTM SE。

我的想法是，解释器到底有没有可能和JIT媲美，解释器最快能有多快？

我知道有许多这类高性能解释器，实际上 LuaJIT的 `-joff` 模式就是杰作，Wasm3 也超快，但我想要自己写一个。
尤其是我接触 iSH 之后，我看了它的源码，大量汇编，我希望我的解释器也能执行 x86 代码，但是我不想要手写汇编。

我从LuaJIT Remake接触到了 clang 的 "新" ABI, `preserve_none`, `preserve_all`, `[[musttail]]`。
对于解释器而言，分发开销或许是核心开销，`preserve_none` + `[[musttail]]` 能生成堪比手写汇编的代码吗？
我不知道，但是我觉得很有趣，值得一试。

而且我刚买了 Gemini 的 Pro 会员，那时候 Google 还算慷慨，我可以用它来写解释器中大量的模版代码。比如从冗长的 Intel 文档提取测试用例，编写实际的指令Handler。
我自己手写了指令分发循环和核心数据结构，然后把 Redis 的 i686 构建反汇编了一遍，让 Gemini 帮我写了个脚本筛出唯一的指令模式，
然后为每种指令模式编写测试用例。

第一周：成功了，跑起了一个 `hello world`，这个程序非常简单，用 int 3 直接调用write系统调用向 stdout 输出字符串，我的模拟器拦截这个指令，输出。

然后我开始用 C# 写 Linux syscall 兼容层，大约花了一个月，跑通了 `CoreMark`。

在项目早期，`CoreMark` 只有大约 `600` 分；经过几轮很具体的优化之后，它最终在我的2023年款M3 Max Macbook Pro稳定到了 `～3000`。并在 iPhone 17 上跑出了 `~3400`

![iPhone 上的 CoreMark](../../assets/images/podish-iphone-coremark.jpeg)

它现在能跑 Busybox，Bash，Python, LuaJIT, GCC，OpenSSH 甚至 Node.js。我成功在上面跑起了 Gemini CLI，但是没多久就会Crash，关闭 V8 的 JIT 能稳定一点，但是性能不可用。

![iPhone 上的 Gemini CLI](../../assets/images/podish-iphone-gemini-cli.jpeg)
![macOS 上的 Gemini CLI](../../assets/images/podish-gemini-cli-macos.png)

仓库里目前大致有几层：

- `libfibercpu`: C++ 写的 IA-32 emulator core
- `Fiberish.Core`: Linux runtime / kernel compatibility layer
- `Fiberish.Netstack`: 基于 [smoltcp](https://github.com/smoltcp-rs/smoltcp)
- `Podish.Core`: 更上层的 container/runtime orchestration
- `Podish.Cli`: 面向实际使用的 CLI
- SwiftUI / Browser 界面

我用 C# 实现了大部份的Syscall模拟工作，因为 C# 的跨操作系统兼容层实现的很不错，满足我的可移植性要求，而且我之前开发 Unity 游戏，C# 符合我的习惯。

```text
Guest i686 Linux Binary
  -> Predecode / Block Builder
  -> Interpreter Dispatch + Semantics
  -> MMU / MicroTLB
  -> Linux Runtime / Syscall Layer
  -> VFS / Process / Signal / Network
  -> Host OS / Browser
```

## 跨语言边界：C++ 核心与 C# 运行时的协作

很多读者第一眼会好奇：为什么解释器核心用 C++，而系统调用层用 C#？这两个世界的边界开销会不会吞掉所有性能优势？

我的答案分为两层：

**第一层是“为什么选 C#”。** 在做这个项目的早期，我需要快速实现大约 200 个 Linux syscall 的语义、VFS 层、网络栈和容器生命周期管理。C# 的跨平台 IO、字符串处理、异步模型和丰富的标准库让我能在一个月内把 `CoreMark` 从 "hello world" 推进到可运行状态。如果全程用 C++，我可能还在写 `std::filesystem` 的封装。

**第二层是“边界开销到底有多大”。** 为了避免 P/Invoke 成为瓶颈，我做了一些关键设计：

- **C API 是最小接口**：`libfibercpu` 暴露的完全是 C 风格 API（`X86_Create`, `X86_Run`, `X86_RegRead` 等），C# 端通过 `LibraryImport` / `DllImport` 调用。每个 `EmuState` 在 C# 端只是一个 `IntPtr`，不会触发托管堆的 GC 压力。
- **大块内存零拷贝**：C# 层不会逐字节通过 P/Invoke 读写访客内存。`bindings.h` 提供了 `X86_ResolvePtrForRead/Write`，直接返回宿主指针；C# 用 `Span<byte>` 或 `MemoryMarshal` 映射后直接操作。
- **回调方向用 GCHandle 锚定**：C++ 侧的 Fault、Interrupt、Log 回调需要持有 C# `Engine` 对象的引用。我通过 `GCHandle.Alloc(this)` 把对象钉在堆上，把指针作为 `userdata` 传给 C++，避免了跨边界时的 GC 移动问题。
- **批量 syscall 优于逐条**：解释器热路径里，guest 程序本身在 C++ 里连续执行成百上千条指令才触发一次 syscall。真正跨边界的次数取决于 guest 的 syscall 密度，而不是指令密度。

实际 profile 显示，在计算密集型负载（如 CoreMark）中，跨语言边界开销连 1% 都占不到；在 IO 密集型负载（如 `git clone`）中，瓶颈在网络和 VFS，也不在 P/Invoke。

## SMC 与 JIT Guest

报告前面我提了一句“跑通 LuaJIT 逼着把写可执行页、失效 block cache 等机制做对”，但这其实是模拟器的难题之一。SMC（Self-Modifying Code）在现代 JIT 引擎里几乎无处不在——V8、LuaJIT、.NET JIT 都会先写一段机器码到内存，再跳转过去执行。

我的实现思路不是“监听整个地址空间的写操作”，而是**复用 MMU 的权限系统**：

- 当某个宿主内存页（Host Page）同时被映射为 **可执行** 和 **可写** 时（通常是 `mmap` 带 `PROT_EXEC|PROT_WRITE` 的 JIT heap），MMU 会在内部把它标记为 `External Alias`。
- 如果该页上已经被预解码并缓存了 `BasicBlock`，MMU 会给这个页的写权限打上 `ForceWriteSlow` 标记。
- 后续任何对该页的写操作，在 TLB refill 时会命中 `ForceWriteSlow`，被迫走慢路径；慢路径里调用 `invalidate_code_cache_page(addr)`，将该页关联的所有 `BasicBlock` 标记为失效。
- 更关键的一点是：解释器执行期间如果检测到当前 EIP 所在页被修改（`ShouldInterceptExecWriteForSmc`），会立即 yield 并切换到单指令安全模式重新执行，避免“正在跑的旧代码”和“刚写的新代码”发生竞争。

这套机制让 LuaJIT 的 `-joff` 模式可以稳定运行，也让 Node.js / V8 能够启动。但可能是因为指令集实现仍有Bug或Linux syscall实现不完整，目前仍偶发 crash——这是我知道的明确限制。

### SMC 机制的细节展开

上面的概述已经说明了思路，但如果你在做类似的设计，下面这段值得一看。核心思想是**把所有检测成本压进 MMU 权限位，避免全局的写监听或反汇编扫描**。

**External Alias 追踪**。`MmuCore` 内部维护了一个 `external_aliases` 映射表（key 是 host page 指针，value 包含 `exec_count`、`write_count` 和关联的 guest page 集合）。当 C# 层通过 `mmap` 把一个 host page 同时映射为 `External + Write + Exec` 时，这个页就会被注册到这张表里。这张表只在页表变更时更新（`mmap`/`mprotect`/`munmap`），热路径里完全不碰。

**ForceWriteSlow 的惰性武装**。真正决定一个外部页是否需要走 SMC 检测的，不是它本身带不带 `Exec` 权限，而是**它上面有没有缓存的 BasicBlock**。`refresh_smc_armed_for_host_page()` 会在两种情况下被调用：

1. 新解码了一个 block，需要检查它所覆盖的 host pages 上是否有可写的外部映射；
2. 外部映射被建立或修改（比如 `mprotect` 加了 `PROT_WRITE`）。

如果 `(exec_count > 0) && (该 host page 上存在 live block)`，就给这个页对应的所有 guest page 表项打上 `ForceWriteSlow`。这个 bit 是**运行时临时标记**，不会持久化、不会被 fork 拷贝，也不出现在页表的深拷贝里。

**写路径拦截链**。当 guest 执行一条写指令时：

```text
write() -> MicroTLB miss -> SoftTLB miss -> resolve_slow()
  -> 查 page directory -> 发现权限含 ForceWriteSlow
  -> 调用 invalidate_code_cache_page(guest_addr)
  -> 将关联 host page 上的所有 BasicBlock 标记为 invalid
  -> 继续完成实际内存写
```

`invalidate_code_cache_page` 不是遍历整个 block cache，而是利用 `page_to_blocks` 反向索引表（host page → block list）做到 O(受影响 block 数) 的局部失效。被标记为 invalid 的 block 会在下次 `X86_Run` 尝试进入时自动重新解码。

**执行期竞争：正在跑的旧代码 vs 刚写的新代码**。仅靠“写的时候失效 block cache”还不够——如果当前 EIP 正好落在被写的页上，解释器 tail-call 跳转到的下一条指令可能已经被改写。为此 `mmu_impl.h` 里有 `ShouldInterceptExecWriteForSmc`：

```cpp
if (state->intercept_exec_write_for_smc && !state->allow_write_exec_page) {
    uint32_t current_page = state->ctx.eip >> 12;
    uint32_t target_page  = addr >> 12;
    if (target_page == current_page || target_page == current_page + 1)
        return true;  // 触发 SMC yield
}
```

一旦命中这个条件，写操作不会立即执行，而是把 `state->smc_write_to_exec` 置位，同时让当前 handler 返回一个特殊的 yield flow。`X86_Run` 的主循环检测到 `smc_write_to_exec` 后，会进入单指令安全模式：

- 把 `allow_write_exec_page = true`，允许**恰好一条** guest 指令写入可执行页；
- 对当前 EIP 只解码并执行一个单指令 block（`max_insts = 1`）；
- 执行完后立即清除 `allow_write_exec_page`，恢复拦截态。

这样保证“写操作”和“跳转到新代码”之间有一个明确的指令边界，不会发生竞争。

**多 Engine 共享下的 TLB 一致性**。当 `clone(CLONE_VM)` 创建新线程时，多个 `Engine` 共享同一个 `MmuCore`，但每个 `Engine` 有自己的 `SoftTLB` 和 `block_lookup_cache`。Engine A 的 `invalidate_code_cache_page` 不会自动刷新 Engine B 的本地 TLB。为此 `MmuCore` 里有一个 `RuntimeTlbShootdownRing`（1024 槽的 ring buffer）：

- 页表变更方把被 flush 的 guest page 写入 ring；
- 其他 Engine 在下次进入 `X86_Run` 时调用 `sync_runtime_tlb_shootdowns()`，消费 ring 里新增的 entry，刷新自己的本地 TLB。

这个 ring 的容量足够覆盖绝大多数单次 mprotect/munmap 的页范围；如果超出容量，直接发一个全量 flush。

我本人测试过的 Guest App，不完全统计:

| 类别 | 代表程序 | 当前状态 | 用例 | 备注 |
| --- | --- | --- | --- | --- |
| shell / base userland | `busybox` / `ash` | 稳定可用 | 有专门的 busybox 构建与运行脚本，跑过 `vim` 这类交互程序 | 命令行交互必备 |
| 更完整的 shell | `bash` | 基本可用 | 已进入 browser rootfs 和日常手工环境 | 主要问题不在 shell 本身，而在更复杂工具链组合 |
| 脚本运行时 | `python3` | 已验证，可稳定跑 benchmark | `python-bench` 已进入主对照表 | 大型复杂项目没试过 |
| JIT / SMC 相关 guest | `LuaJIT` | 已验证，可稳定跑 benchmark | `luajit-bench` 已进入主对照表 | 逼着我把写可执行页、失效 block cache 等机制做对 |
| 构建工具链 | `gcc` / `make` | 已验证 | CoreMark tree 上的 `make compile` 跑通 | 没试过编译大型代码库 |
| 网络 / 开发工具 | `git` / `OpenSSH` | 手工跑通过 | `git clone` | 暴露 VFS、PTY、网络边界问题 |
| 重型现代 runtime | `Node.js` | 能启动，但目前不太实用 | 手工 smoke | npm能装包，偶尔 crash，关闭 V8 JIT 后会稳定，但太慢 |
| AI CLI | `Gemini CLI` | 能拉起，但很快会 crash | 手工 smoke | Node/V8 不能稳定运行，卡在这里 |

![iPhone 上的 Vim](../../assets/images/podish-iphone-vim.jpeg)
![iPhone 上的 Yazi](../../assets/images/podish-iphone-yazi.jpeg)

## 对 Copy-and-patch JIT 的尝试

Copy-and-patch 这种 JIT 的核心思路很简明：先把 Opcode handler 预编译成 binary stencil，然后在运行时只做“拷贝模板 + patch 常量/地址/跳转槽位”，从而避开完整后端、寄存器分配和传统机器码生成器的复杂度。 可以看论文 [Copy-and-Patch Compilation](https://arxiv.org/abs/2011.13127)。

我用 Clang/LLVM 的那套 ABI 特性来优化解释器热路径，比如 `preserve_none`、`preserve_all`、`[[musttail]]`。如果这些属性已经能让 handler 很接近手写汇编风格，那么“把 handler 变成 stencil 再 patch 参数”也不难。相关属性文档都在 Clang 的 [Attribute Reference](https://clang.llvm.org/docs/AttributeReference.html) 里。

结构很简单，我已经有了每个指令的 Handler，编译出 Stencil，把取指令中参数的代码直接 Patch 成取立即数，然后连接在一起。

我本来期待 200%+ 的性能提升，实际上比解释器还略慢。

现在回头看，这个结果并不奇怪。Copy-and-patch 真正省掉的是“解释器反复 decode / dispatch 同一段逻辑”的那部分成本；但我这边的 direct-threaded interpreter 已经把 dispatch 压得很低了，反而是访存、地址转换、状态维护、I-Cache 压力这些东西成了瓶颈。于是 stencil 虽然省掉了一些字节码读取，但是 patch 后由于代码膨胀，访存压力，尤其是 I-Cache 压力反而更大了。

这件事迫使我 profile：

- 在 direct-threaded interpreter 已经把分发成本压低之后，瓶颈未必还在 dispatch
- 对当前这个设计来说，访存和地址转换链路比我想象中更贵
- 劣质 JIT 是垃圾，I-Cache 更紧张了，加载立即数也不会比加载字节码快

## 核心思路 1：预解码 + tail-call dispatch

我设计的路线：

- 先把 x86 指令预解码成固定长度中间表示
- 基本块内顺序存放
- 每条 IR 都带上下一条访客 PC、操作数描述和执行入口
- 执行完之后不回统一 dispatcher，而是直接 tail-call 到下一条语义入口

这样做的目标：

- 把变长 x86 解码尽量前移，并且缓存解码结果
- 减少主循环分支和调度开销
- 让热路径更适合 host ABI 和编译器优化

非常经典的思路，不过效果非常好，令我惊讶的是它比 QEMU TCI 快 10X。

感谢 Justine Tunney 的 [blink](https://github.com/jart/blink) 项目，我靠它了解到了 Intel 的 [XED](https://github.com/intelxed/xed)。
并最终得到了表驱动的 i686 解释器。

一条预解码后的中间表示大致长这样：

`DecodedOp` 固定 **32 字节**，对齐到 16 字节边界：

| Field | Type | Offset | Size | 说明 |
| --- | --- | ---: | ---: | --- |
| `handler` | `HandlerFunc*` (函数指针) | 0 | 8 | 执行该指令语义的入口 |
| `next_eip` | `uint32_t` | 8 | 4 | 下一条指令的 PC |
| `len` | `uint8_t` | 12 | 1 | 指令字节长度 |
| `modrm` | `uint8_t` | 13 | 1 | ModRM 字节 |
| `prefixes` | `Prefixes` (union) | 14 | 1 | 前缀信息（lock/rep/segment…） |
| `meta` | `Meta` (union) | 15 | 1 | 元信息（has_mem / has_imm / ext_kind…） |
| `ext.data` | `DecodedMemData` | 16 | 16 | 内存操作数描述（imm / ea_desc / disp） |
| `ext.link.next_block` | `BasicBlock*` | 24 | 8 | block linking 后缓存的后继块指针 |
| `ext.control` | `DecodedControlFlowData` | 16 | 16 | 控制流目标（target_eip / cached_target） |

如果拿一条简单内存 ALU 指令举例：

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

## 核心思路 2：Parity-only lazy flags

x86 的 `EFLAGS` 实现比较繁琐，我看了 QEMU 有一套非常完善的 [lazy flags 实现](https://qemu.weilnetz.de/w64/2012/2012-12-04/qemu-tech.html)。

>   2.3 Condition code optimisations
>   Lazy evaluation of CPU condition codes (EFLAGS register on x86) is important for CPUs where every instruction sets the condition codes. It tends to be less important on conventional RISC systems where condition codes are only updated when explicitly requested. On Sparc64, costly update of both 32 and 64 bit condition codes can be avoided with lazy evaluation.
>
>   Instead of computing the condition codes after each x86 instruction, QEMU just stores one operand (called CC_SRC), the result (called CC_DST) and the type of operation (called CC_OP). When the condition codes are needed, the condition codes can be calculated using this information. In addition, an optimized calculation can be performed for some instruction types like conditional branches.
>
>   CC_OP is almost never explicitly set in the generated code because it is known at translation time.
>
>   The lazy condition code evaluation is used on x86, m68k, cris and Sparc. ARM uses a simplified variant for the N and Z flags.

我尝试实现过一版，性能比直接算还要慢。直接原因是需要把 `CC_SRC`, `CC_DST`, `CC_OP` 都写入内存，实际求值时又要从内存读，我的程序已经是 Memory-bound，这样做没意义。

我最后取其中，不全 Lazy，只 Lazy PF。具体来说，解释器执行期间传递的 `flags_cache` 不是 `uint32_t eflags`，而是一个 `uint64_t`：低 32 位就是 architectural EFLAGS 的实时映像，高位（当前用第 40-47 位）存一个 parity state byte。

```text
flags_cache (uint64_t)
  [31:0]  = architectural EFLAGS (CF/PF/AF/ZF/SF/OF/DF... 实时维护)
  [47:40] = PF lazy state (source byte, 不是 PF bit 本身)
```

这样设计的原因是：CF/ZF/SF/OF 几乎每个 ALU 指令都要写，而且 `jcc`/`cmov`/`setcc` 几乎每条都要读，如果做 lazy，每次写和每次读都要额外存/取 `CC_SRC`/`CC_DST`/`CC_OP`，我已经是 Memory-bound，不如直接算。但 PF 的求值需要遍历结果低 8 位的 parity，比普通 flags 贵一个数量级，而且 `jcc` 中用到 PF 的只有 `JP`/`JNP`（cond 10/11），比例很低，值得 lazy。

**静态层**：在基本块内做 flags 的 def-use 分析。如果某条指令写 flags，但后面没人读，就直接换成 no-flags Handler 变体。这种变体在 `AluAdd<T, false>`、`AluSub<T, false>` 等模板参数中体现，编译器会把整个 flags 更新路径完全剪掉。

**动态层**：每条指令执行时，`flags_cache` 作为寄存器参数在 tail-call 链中传递，永不写回内存。只有 `PF` 是 lazy 的：

- 写 PF 时，调用 `SetParityState(flags_cache, result_byte)`，把结果低 8 位直接塞进高位 parity state，不计算 parity；
- 读 PF 时，调用 `EvaluatePF(flags_cache)`，从高位取出 source byte，现场算 parity；
- 外部可见点（`pushf`、`lahf`、fault、interrupt、API boundary）调用 `GetArchitecturalEflags()`，它会 materialize lazy parity 后再返回完整的 32 位 EFLAGS。

**Commit 语义**：`CommitFlagsCache` 只在 handler chain 退出边界调用一次（`ExitOnCurrentEIP`、`ExitOnNextEIP`、restart/retry、resolver miss），把寄存器中的 `flags_cache` 写回 `state->ctx.flags_state`。成功 chaining 的情况下绝不 commit——因为下一条指令的 handler 还会以寄存器参数的形式继续传递同一个 `flags_cache`。`X86_Run()` 和 `X86_Step()` 在 handler 返回后也不再 commit，避免用 stale 的 caller-side copy 覆盖已更新的状态。

一个有意思的细节是 `CheckCondition` 的 LUT 路径：绝大多数条件跳转（cond 0-9, 12-15）不依赖 PF，所以它们走 `GetFlags32Raw(flags_cache)` 直接读取低 32 位，用 LUT 查表决定跳转方向；只有 cond 10/11（JP/JNP）会单独分支到 `EvaluatePF()`。这样设计让 PF lazy 的额外成本几乎为零。

## 核心思路 3：访存是瓶颈

项目开始，我一直在分析 dispatch 开销，毕竟我的核心思路就是降低快路径成本，用 Threading 降低 dispatch 开销。

第一次 profile 才发现，真正的问题甚至更基础：TLB看起来完整没有命中，大量的开销在Page table walk。

`SoftTLB` 是一个三路 direct-mapped TLB，三张固定大小的表：

`SoftTlb` 由三张固定大小的 direct-mapped 表组成：

| Field | Type | Offset | Size | 说明 |
| --- | --- | ---: | ---: | --- |
| `read_tlb` | `std::array<TlbEntry, 256>` | 0 | 4096 | 读权限快速映射表 |
| `write_tlb` | `std::array<TlbEntry, 256>` | 4096 | 4096 | 写权限快速映射表 |
| `exec_tlb` | `std::array<TlbEntry, 256>` | 8192 | 4096 | 执行权限快速映射表 |

`TlbEntry` 对齐到 16 字节，固定 **16 字节**：

| Field | Type | Offset | Size | 说明 |
| --- | --- | ---: | ---: | --- |
| `tag` | `uint32_t` | 0 | 4 | guest page 的 tag（高 20 位） |
| `perm` | `Property` (enum) | 4 | 4 | 权限位（Read / Write / Exec / Dirty …） |
| `addend` | `std::uintptr_t` | 8 | 8 | `host_ptr = guest_addr + addend` |

索引方式也非常直接：

```text
idx = (guest_addr >> PAGE_SHIFT) & 255
tag = guest_addr & ~PAGE_MASK
host_ptr = guest_addr + addend
```

也就是说，`SoftTLB` 的职责是把已经查过的 guest page 映射压成一个 `tag + addend` 的快速表项。命中的时候，只要检查 tag，再做一次加法就能拿到宿主地址；miss 的时候，才回到慢路径检查权限、补表项、处理异常。

原因最后是一个很低级 TLB refill 错误。修掉之后，`CoreMark` 先从 `600` 提到了 `800`。

它让我意识到： **地址转换路径会是瓶颈**

于是我往这个方向做了进一步设计：

- 在执行链路中携带极小的 `MicroTLB` 常驻寄存器
- 命中后做 tag match + addend 加法
- 调整热字段布局，尽量让编译器更容易生成成对加载
- 在少数关键路径上强制把相关字段成组提前加载

`MicroTLB` 的核心结构其实很简单：

`MicroTLB` 常驻寄存器，对齐到 16 字节，固定 **16 字节**：

| Field | Type | Offset | Size | 说明 |
| --- | --- | ---: | ---: | --- |
| `tag_r` | `uint32_t` | 0 | 4 | 读权限 tag，默认 `0xFFFFFFFF`（未命中态） |
| `tag_w` | `uint32_t` | 4 | 4 | 写权限 tag，默认 `0xFFFFFFFF`（未命中态） |
| `addend` | `std::uintptr_t` | 8 | 8 | `host_ptr = guest_addr + addend` |

注意数据布局，read_tag + write_tag 合并在同一个寄存器，所以这个结构占两个 Host 寄存器。

它的 hit / miss 路径可以用下图表示：

```text
guest address
  -> check read_tag / write_tag
  -> hit: host_ptr = guest_addr + host_guest_addend
  -> miss: full translation + permission check + refill MicroTLB
```

refill 的时候要检查权限，如果没有 read 权限，清除 read_tag，反之亦然。
这个设计可能有点奇怪，如果读写反复不在同一个页面上，MicroTLB 会 Ping-pong，命中率为0。
但大多数时候有效，我测下来命中率高于50%，实际上只要有一定的命中，能降低从内存的 SoftTLB 读的频率，就有性能提升。

## 核心思路 4：block linking 和 superopcode

在这个解释器里，`BasicBlock` 是内存里的定长头部 + 变长指令流对象。它的头部大致长这样：

`BasicBlock` 对齐到 16 字节，头部固定 **48 字节**，后跟变长的 `DecodedOp` 数组：

| Field | Type | Offset | Size | 说明 |
| --- | --- | ---: | ---: | --- |
| `chain.start_eip` | `uint32_t` | 0 | 4（嵌入 `BasicBlockChainPrefix`） | 块起始 guest PC |
| `chain.inst_count` | `uint8_t` | 4 | 1 | 块内指令数 |
| `chain.valid` | `uint1_t` | 5 | 1 bit | 块是否有效（用于失效标记） |
| `entry` | `HandlerFunc*` | 8 | 8 | 块首条指令的语义入口，解释器直接跳转到这里 |
| `end_eip` | `uint32_t` | 16 | 4 | 块结束地址（最后一条指令的 next_eip） |
| `slot_count` | `uint32_t` | 20 | 4 | 总 slot 数（含末尾 sentinel） |
| `sentinel_slot_index` | `uint32_t` | 24 | 4 | sentinel 在 slots 中的索引 |
| `branch_target_eip` | `uint32_t` | 28 | 4 | 分支目标地址（条件跳转/直接跳转时有效） |
| `fallthrough_eip` | `uint32_t` | 32 | 4 |  fallthrough 地址 |
| `terminal_kind_raw` | `uint8_t` | 36 | 1 | 终止类型（None / DirectJmpRel / DirectJccRel / Other） |
| `block_padding0` | `uint8_t` | 37 | 1 | 填充 |
| `block_padding1` | `uint16_t` | 38 | 2 | 填充 |
| `exec_count` | `uint64_t` | 40 | 8 | 块被执行的次数（用于 profile-guided superopcode） |
| `slots[]` | `DecodedOp[]` | 48 | 变长（每个 32 字节） | 预解码后的指令流 |

其中有几个字段很关键：

- `entry` 指向这个块首条可执行语义入口，解释器拿到块之后可以直接从这里开始跑
- `slots[]` 是按顺序排好的 `DecodedOp` 数组，每个 `DecodedOp` 固定 `32` 字节
- `slot_count` 包含末尾 sentinel，所以块的内存布局是固定头部加一段连续的 op stream
- `branch_target_eip` / `fallthrough_eip` 让 block linking 可以在块级别预先知道后继去向
- `exec_count` 则是 profile-guided superopcode 的重要输入

`DecodedOp` 自己也不是“只有 handler 指针”的极简结构。它除了 `handler` 和 `next_eip` 之外，还会在扩展区里保存内存操作数信息、控制流目标，或者 block linking 之后缓存下来的 `next_block` 指针。

所以 `BasicBlock` 真正做的事情，其实是把三件事绑在一起：

- 缓存预解码结果，避免重复 decode
- 把一串 `DecodedOp` 紧凑地放成可 tail-call 的执行流
- 给 block linking / superopcode / profile 这些后续优化提供稳定载体

我后面做 block linking 时，在现有 `BasicBlock` 上直接做拼接和复用。

### Block linking

如果一个基本块足够短、指令不跨页，而且控制流关系简单，就把后继块直接接到当前块尾部。
这减少了块间跳转时的一部分间接访存。

### Superopcode

一开始我试过简单 bigram 统计，但效果一般。原因显而易见，高频 Bigram 的结果不一定就是被频繁执行，只是这一对在所有对里的频率高，但是本身固定的指令对就不多。
另外我们做了指令特化，特化之后的指令变得极其稀疏，能固定的 pair 就更少了。

更有效的做法是：

- 先围绕高频 anchor instruction 选种子
- 再看它和前后指令的 def-use 关系
- 只融合存在 `RAW` 依赖的组合

这个策略最终生成了大约 `256` 个 superopcode，也是性能第一次稳定跨过 `3000+ CoreMark` 的关键步骤之一。

当前生成集里，比较有代表性的例子包括：

- 栈操作链：`pop ebx ; pop esi`
- flags 生产者/消费者链：`test eax, eax ; je/jz ...`
- load-use 链：`mov eax, [esp+off] ; sub reg, eax`
- load-store 链：`mov ebx, [esp+off] ; mov [esi], ebx`

候选发现流程本质上也不复杂：

```text
block trace
  -> 热点统计
  -> anchor 选择
  -> def-use / RAW 过滤
  -> 代码生成
  -> 回归验证与收益复核
```

## 从 600 到 3000：Profile 与优化经历

- 随着优化的进行，瓶颈会快速迁移
- 教科书级别的实现，并不一定贴合特定场景
- 一个设计足够贴近硬件的解释器，未必比一个垃圾的 Baseline JIT 差

| 阶段 | 关键变化 | CoreMark | 备注 |
| --- | --- | ---: | --- |
| 初始版本 | baseline interpreter | ~600 | ABI / TLB 设计已经在，但 TLB bug 让地址转换热路径几乎完全失效 |
| 修 TLB bug | fix address translation path | ~800 | 先确认地址转换是早期最大问题 |
| 热路径优化 | PC / block budget / template specialization | ~1500 | 不再每条指令都做多余状态写内存，解释器开始接近可用区间 |
| 访存整理 | data layout / paired loads | ~2000 | 优化重点从 dispatch 转向访存组织 |
| lazy flags | static prune + PF lazy | ~2200 | 全 lazy flags 试验版更慢，选择目前的中庸方案 |
| block linking | append simple successor block | ~2500 | 降低块边界成本 |
| superopcode | profile-guided fused ops | ~3000 | 约 `256` 个 anchor + RAW superopcode 把结果稳定在 `~3000` |

## 两个“反常识”的数据现象

### A19 比 M3 Max 还快？

在这个项目里，解释器是极度依赖单线程 IPC 和访存延迟的工作负载，几乎吃不到多核和内存带宽的红利。M3 Max 的 300GB/s 内存带宽和 A19 相比，在 CoreMark 这种小工作集场景下拉不开差距。所以 A19 小幅领先是正常的代际差异。

### iSH 上 `luajit -joff` 比开启 JIT 还快

另一个让我困惑的现象：在 iSH 上，`luajit -joff`（纯解释执行）跑 `primes_jit.lua` 耗时 27530 ms，而开启 JIT 的 `luajit` 要 46650 ms。

这完全违背了“JIT 应该更快”的直觉。我目前的推测是：

- iSH 的实现方式（基于 `i386` 的完整系统模拟）对 SMC 的处理路径非常重，LuaJIT 开启 JIT 后会频繁生成和修补机器码，导致 iSH 的代码缓存频繁失效、flush、重翻译。
- `-joff` 模式下 LuaJIT 退化成纯 bytecode 解释器，不再生成机器码，也就不会触发 iSH 的 SMC 陷阱，整体反而更流畅。
- 另一个可能是 iSH 的 TLB / 页表 walk 在写保护页（用于 SMC 检测）上存在额外开销。

> 我对 iSH 内部实现没有深入研究过，以上只是猜测。如果你熟悉 iSH 的源码，非常欢迎在评论区告诉我真正的成因。

## 多线程模型：单调度器线程 + 协作式任务切换

我实现了 `clone`/`fork`/`vfork` 和基本的 `pthread` 语义，但**并发模型并不是操作系统意义上的多线程并行**。

具体架构如下：

- `KernelScheduler` 绑定到一个固定的宿主线程（scheduler thread）。所有 `FiberTask`（对应 Linux 的 task_struct）的创建、切换、系统调用分发和信号递送，都在这个线程上顺序执行。
- 每个 `FiberTask` 拥有自己的 `Engine` 实例（也就是独立的 `EmuState`），因此解释器核心本身**不需要**线程安全设计——不存在两个线程同时读写同一个 `EmuState` 的情况。
- 当 guest 程序调用 `clone(CLONE_VM | CLONE_THREAD)` 创建新线程时，C# 层会新建一个 `FiberTask` 和新的 `Engine`，但让两个 `Engine` 共享同一个 `MmuCore`（通过原子引用计数管理生命周期）。
- 共享 `MmuCore` 带来了一个问题：Engine A 修改了页表或刷新了 block cache，Engine B 的 `SoftTLB` 和 `block_lookup_cache` 仍然是旧的。为此我实现了一个轻量的 `RuntimeTlbShootdownRing`：页表变更方把受影响的 guest page 写入一个 ring buffer，其他 Engine 在下次执行前同步消费这个 ring，刷新自己的本地 TLB。

这种设计的上限是： guest 的多个线程在**宿主层面是串行调度的**，不能利用多核并行。对于容器里的 shell、Python、编译任务来说，这通常足够；但如果 guest 跑的是重度并行的科学计算，性能会受到明显限制。

## 现在能跑啥

除了 CPU 核心之外，这个项目还实现了一层 Linux 兼容环境，包括：

- 类 Linux 的 `fork` / `vfork` / `clone` / `execve` / `wait*` 语义
- `tmpfs` / `procfs` / host mounts / overlay roots
- PTY / TTY
- sockets 和原生 netstack 集成
- OCI image pull/save/load/import/export
- 以 `Podish.Cli` 为入口的 container-style execution

这部分其实决定了它能不能从 benchmark 走向真实工作负载。

| 类别 | 代表程序 / 证据 | 当前状态 | 主要限制 | 备注 |
| --- | --- | --- | --- | --- |
| shell | `/bin/sh -lc true` | 已验证 | 目前正文主要覆盖 shell 启动和短命令 | |
| coreutils / archive | `grep -R` / `tar cf` / `tar xf` | 已验证 |  |  |
| scripting | `python` (`primes.py`) | 已验证 | workload 当前只有少量样本，容易受手机上温度影响 |  |
| JIT-heavy | `LuaJIT` (`primes_jit.lua`) | 已验证 | 结果曾被 notify socket / 网络路径 bug 污染，已修复 | JIT 相关 guest 能跑 |
| build-oriented mixed workload | `make compile` on CoreMark tree | 已验证 | 当前主表按 compile-only 口径统计；每轮先 `make clean`，再计字面 `make compile` | |
| network / tooling compat | `git clone` | 兼容性已验证 | 网络波动大 | 主要验证 VFS / DNS / TLS / git 链路 |

## 当前已跑出的第一组对照数据

- Podish / QEMU 当前主表里的桌面数据来自 `MacBook Pro (Apple M3 Max, macOS 26.3)`
- iSH 当前数据来自 `iPhone 17` 标准版（`A19`），环境是 `Alpine Linux v3.14.3`
- Podish / QEMU Guest 目前采用 `docker.io/i386/alpine:latest`
- native host 参考值也来自同一台 `MacBook Pro (Apple M3 Max, macOS 26.3)`

### 启动速度和计算密集任务

| Workload | Podish(A19) | Podish(M3 Max) | iSH (A19) | Podman i386 (QEMU JIT) | QEMU TCI | Native host (M3 Max) | 测试说明 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| `sh -lc true` warm start | 20 ms | 20 ms | 30 ms | 30 ms | 30 ms | 10 ms | 这一行是端到端启动成本；Podish 用 `Podish.Cli run docker.io/i386/alpine:latest /bin/sh -lc true`；QEMU 列为显式 `qemu-i386 -L <rootfs>`；native 是宿主 `/bin/sh` |
| CoreMark 1.0 | 3447 | 2967 | 1692 | 11456 | 325 | 38087 | |
| `python primes.py`| 78.3 s | 89.4 s | 684.4 s | 40.9 s | 787.3 s | 1.8 s | 
| `luajit primes` | 3.1 s | 4.0 s | 46.6 s | 1.7 s | 39.3 s | 0.2 s |
| `luajit -joff` | 14.5 s | 17.3 s | 27.5 s | 7.1 s | 152.7 s | 0.7 s |

脚本语言测试用例来自 [kostya的跨语言Benchmark](https://github.com/kostya/benchmarks)

这些数据似乎显示 iSH 对脚本语言有奇怪的行为，甚至 `luajit -joff` 比开启 JIT 快，我还没有研究出为什么。

另外，显然对于模拟器，虽然 CoreMark 看着还可以，跑复杂应用明显性能掉的厉害，推测是地址转换导致的问题。

我在同一台 Macbook 上直接用 `wasm3 coremark.wasm` 跑了 5 次，作为另一个解释器的参考。

- 第 1 次是 `Iterations = 60000`，`CoreMark = 3308.108090`
- 后 4 次稳定在 `Iterations = 40000`，分数分别是 `3292.879486`、`3246.276318`、`3212.163048`、`3208.350438`
- 中位数约为 `3229.22`

也就是说，Podish 的纯计算性能已经可以对比 Wasm3 了。

### 文件IO和混合任务

| Workload | Podish(A19) | Podish(M3 Max) | iSH (A19) | Podman i386 (QEMU JIT) | Native host (M3 Max, arm64) | 备注 |
| --- | ---: | ---: | ---: | ---: | --- |
| `grep -R` on CoreMark tree | 50 ms | 50 ms | 70 ms | 70 ms | 10 ms | 排除 `.git`, GNU grep |
| `tar cf` CoreMark tree | 40 ms | 30 ms | 40 ms | 97 ms | 10 ms | GNU tar |
| `tar xf` CoreMark tree | 250 ms | 250 ms | 170 ms | 161 ms | 30 ms | GNU tar |
| `make compile` on CoreMark tree | 9470 | 10460 ms | 11430 | 7330 ms | 210 ms | `make compile`；native 为单进程 `make -j1 compile CC=clang` |
| `git clone` CoreMark | 3230 ms | 3300 ms | 5190 ms | 2660 ms | 2980 ms | 只做兼容性验证，不作为性能结论；网络波动大 |

QEMU TCI 模式暂缺，太慢了，并且在这里意义不大。

本机 clang 快得离谱，这也说明 CoreMark 不是全部。

以上两张表，除了有明确说明的以外，均为5次实验的中位数。

## 图形与音频：Wayland 和 PulseAudio 的实验性支持

文章通篇都在讲命令行和 benchmark，但其实代码库里已经有了两条实验性的多媒体管线，只是它们还没有成熟到进入日常测试。

- **`Podish.Wayland`**（约 7K 行 C#）：实现了一个轻量 Wayland 合成器核心，能把 guest 的 Wayland 客户端协议桥接到宿主 SDL 窗口。CLI 里有 `WaylandSdlDisplayHost` 和 `WaylandSdlFramePresenter`，意味着你理论上可以在 macOS 桌面上开一个窗口，跑 Linux 的 GUI 程序。
- **`Podish.Pulse`**（约 6K 行 C#）：实现了 PulseAudio 协议栈的客户端/服务器端，能把 guest 的音频流重定向到宿主的音频后端。CLI 里同样有 `PodishPulseVirtualDaemon` 在集成。

目前这两条线的状态是“代码存在、架构清晰、但缺少端到端的打磨”。图形路径目前只支持通过 shmem 发送Wayland Surface，不支持 gbm/EGL，我试了foot能跑通，但是SDL2还不行，似乎Wayland上的SDL2强制要求EGL；音频路径目前比较粗糙，不过 ffplay -nodisp 能播放音乐。

![Wayland Foot 终端演示](../../assets/images/podish-wayland-foot.png)

这两个功能目前只在macOS上支持，iOS / Wasm还没做适配。
