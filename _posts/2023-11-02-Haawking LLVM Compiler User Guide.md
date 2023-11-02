---
layout: post
title: Haawking LLVM Compiler User Guide
date: 2023-11-02
updated: 2023-11-02
categories: Tech
tags: [Essay，Tech]

description: 优化分为高层次和低层次两个大类，高层次优化就是通常所说的O1或者O2，而低层次优化就是由代码生成器（Code Generator）来完成，跟具体的芯片架构相关。
---

## 1.编译器优化选项

优化分为高层次和低层次两个大类，高层次优化就是通常所说的O1或者O2，而低层次优化就是由代码生成器（Code Generator）来完成，跟具体的芯片架构相关。

### 1.1 CG2000编译器优化选项

作为DSP领域最常见的开发工具，在TI的CCS中，cg2000编译器共有6级优化选项，off、0、1、2、 3和4，具体含义如下： 

| 编译选项                 | 具体含义                                                     |
| ------------------------ | :----------------------------------------------------------- |
| --opt_level=off or -Ooff | Performs no optimization                                     |
| --opt_level=0 or -O0     | Performs control-flow-graph simplification <br />Allocates variables to registers<br /> Performs loop rotation Eliminates unused code <br />Simplifies expressions and statements <br />Expands calls to functions declared inline |
| --opt_level=1 or -O1     | Performs all --opt_level=0 (-O0) optimizations, plus: <br />Performs local copy/constant propagation <br />Removes unused assignments<br /> Eliminates local common expressions |
| --opt_level=2 or -O2     | Performs all --opt_level=1 (-O1) optimizations, plus: <br />Performs loop optimizations<br /> Eliminates global common subexpressions <br />Eliminates global unused assignments <br />Performs loop unrolling |
| --opt_level=3 or -O3     | Performs all --opt_level=2 (-O2) optimizations, plus:   <br />Removes all functions that are never called <br />Simplifies functions with return values that are never used <br />Inlines calls to small functions<br /> Reorders function declarations; the called functions attributes are known when the caller is optimized <br />Propagates arguments into function bodies when all calls pass the same value in the same argument position <br />Identifies file-level variable characteristics |
| --opt_level=4 or -O4     | Performs link-time optimization.                             |
|                          |                                                              |

### 1.2 Haawking LLVM Compiler编译选项

Haawking LLVM Compiler V1版本，基于LLVM13进行开发和优化，因此兼容基本的优化选项；同时，昊芯增加了一个Odefault编译选项，优化等级介于O0和O1之间。

| 编译选项   | 具体含义                                                     |
| ---------- | ------------------------------------------------------------ |
| -O0/-Otest | Only essential optimizations (e.g. inlining “always inline” functions) |
| -Odefault  |                                                              |
| -O1        | Optimize quickly while losing minimal debugging information. |
| -O2        | Apply all optimization. Only apply transformations practically certain to produce better performance. |
| -O3        | Apply all optimizations including optimizations only likely to have a beneficial effect (e.g. vectorization with runtime legality checks). |
| -Os        | Same as -O2 but makes choices to minimize code size with minimal performance impact. |
| -Oz        | Same as -O2 but makes choices to minimize code size regardless of the performance impact. |
|            |                                                              |

### 1.3 简单示例

对于下面这段简单的示例代码，我们可以看一下O0和O1之间的差别。

```c
int foo(int var) {
    return var * 10;
}
int main() {
    int test1, test2;
    test1 = 20;

    test2 = foo(test1);

    return 0;
}
```

当我们采用O0编译选项编译这段代码的时候，可以得到最基本的程序代码，如下所示（编译工具选择Compiler Explorer上面RISC-V rv32gc clang 13.0.1）： 

```assembly
foo(int):                                # @foo(int)
        addi    sp, sp, -16
        sw      ra, 12(sp)                      # 4-byte Folded Spill
        sw      s0, 8(sp)                       # 4-byte Folded Spill
        addi    s0, sp, 16
        sw      a0, -12(s0)
        lw      a0, -12(s0)
        li      a1, 10
        mul     a0, a0, a1
        lw      ra, 12(sp)                      # 4-byte Folded Reload
        lw      s0, 8(sp)                       # 4-byte Folded Reload
        addi    sp, sp, 16
        ret
main:                                   # @main
        addi    sp, sp, -32
        sw      ra, 28(sp)                      # 4-byte Folded Spill
        sw      s0, 24(sp)                      # 4-byte Folded Spill
        addi    s0, sp, 32
        li      a0, 0
        sw      a0, -24(s0)                     # 4-byte Folded Spill
        sw      a0, -12(s0)
        li      a0, 20
        sw      a0, -16(s0)
        lw      a0, -16(s0)
        call    foo(int)
        mv      a1, a0
        lw      a0, -24(s0)                     # 4-byte Folded Reload
        sw      a1, -20(s0)
        lw      ra, 28(sp)                      # 4-byte Folded Reload
        lw      s0, 24(sp)                      # 4-byte Folded Reload
        addi    sp, sp, 32
        ret
```

而当我们使用O1编译的时候，则可以得到优化后的如下代码：

```assembly
foo(int):                                # @foo(int)
        li      a1, 10
        mul     a0, a0, a1
        ret
main:                                   # @main
        li      a0, 0
        ret
```

而当我们使用Haawking IDE中集成的Haawking LLVM Compiler提供的Odefault编译选项编译时，可以得到优化后的代码如下，其优化力度介于O0和O1之间：

```
foo(int):                                # @foo(int)
        addi    sp, sp, -16
        addi    a1, zero, 10
        mul     a1, a0, a1
        sw      a0, 12(sp)
        mv      a0, a1
        addi    sp, sp, 16
        ret
main:                                   # @main
        addi    sp, sp, -16
        sw      ra, 12(sp)                      # 4-byte Folded Spill
        sw      zero, 8(sp)
        addi    a0, zero, 20
        sw      a0, 4(sp)
        addi    a0, zero, 20
        call    foo(int)
        mv      a0, zero
        lw      ra, 12(sp)                      # 4-byte Folded Reload
        addi    sp, sp, 16
        ret
```

为了更加直观地对比三种编译选项的差异，将foo函数的反汇编代码进行汇总，如下表所示：

| 编译选项O0                                                   | 编译选项Odefault                                             | 编译选项O1                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| foo(int):<br/>        addi    sp, sp, -16<br/>        sw      ra, 12(sp)<br/>        sw      s0, 8(sp)<br/>        addi    s0, sp, 16<br/>        sw      a0, -12(s0)<br/>        lw      a0, -12(s0)<br/>        li      a1, 10<br/>        mul     a0, a0, a1<br/>        lw      ra, 12(sp)<br/>        lw      s0, 8(sp)<br/>        addi    sp, sp, 16<br/>        ret | foo(int):  <br/>        addi    sp, sp, -16<br/>        addi    a1, zero, 10<br/>        mul     a1, a0, a1<br/>        sw      a0, 12(sp)<br/>        mv      a0, a1<br/>        addi    sp, sp, 16<br/>        ret | foo(int):  <br/>        li      a1, 10<br/>        mul     a0, a0, a1<br/>        ret |
| 48Bytes                                                      | 28Bytes                                                      | 12Bytes                                                      |
|                                                              |                                                              |                                                              |

通过Compiler Explorer，可以将有效的pass进行过滤并输出，三种编译选项的pass见下表所示：

| 编译选项O0                                                   | 编译选项Odefault                                             | 编译选项O1                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| <br /><br /><br /><br /><br />RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/><br /><br /><br /><br /><br /><br />Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Fast Register Allocator (regallocfast)<br/><br /><br /><br /><br /><br /><br /><br /><br /><br />Prologue/Epilogue Insertion & Frame Finalization (prologepilog)<br/><br /><br /><br />RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/><br /><br /><br /><br /><br /><br /><br /><br /><br />Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Fast Register Allocator (regallocfast)<br/><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br />Prologue/Epilogue Insertion & Frame Finalization (prologepilog)<br/>Post-RA pseudo instruction expansion pass (postrapseudos) | <br /><br /><br /><br /><br />RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/>Slot index numbering (slotindexes)<br/>Merge disjoint stack slots (stack-coloring)<br/>Live Variable Analysis (livevars)<br/>Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Slot index numbering (slotindexes)<br/>Live Interval Analysis (liveintervals)<br/>Machine Instruction Scheduler (machine-scheduler)<br/>Virtual Register Rewriter (virtregrewriter)<br/>Machine Copy Propagation Pass (machine-cp)<br/>Prologue/Epilogue Insertion & Frame Finalization (prologepilog)<br/>Post-RA pseudo instruction expansion pass (postrapseudos)<br/>RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/>Slot index numbering (slotindexes)<br/>Merge disjoint stack slots (stack-coloring)<br/>Remove dead machine instructions (dead-mi-elimination)<br/>Live Variable Analysis (livevars)<br/>Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Slot index numbering (slotindexes)<br/>Live Interval Analysis (liveintervals)<br/>Simple Register Coalescing (simple-register-coalescing)<br/>Greedy Register Allocator (greedy)<br/>Virtual Register Rewriter (virtregrewriter)<br/>Machine Copy Propagation Pass (machine-cp)<br/>Prologue/Epilogue Insertion & Frame Finalization (prologepilog)<br/>Post-RA pseudo instruction expansion pass (postrapseudos) | SROA on foo(int)<br/>SROA on main<br/>GlobalOptPass on [module]<br/>InlinerPass on (main)<br/>RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/>Slot index numbering (slotindexes)<br/>Merge disjoint stack slots (stack-coloring)<br/>Live Variable Analysis (livevars)<br/>Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Slot index numbering (slotindexes)<br/>Live Interval Analysis (liveintervals)<br/><br /><br /><br />Virtual Register Rewriter (virtregrewriter)<br/>Machine Copy Propagation Pass (machine-cp)<br/><br /><br /><br /><br /><br /><br />RISCV DAG->DAG Pattern Instruction Selection (amdgpu-isel)<br/>Slot index numbering (slotindexes)<br/>Merge disjoint stack slots (stack-coloring)<br/><br /><br /><br />Live Variable Analysis (livevars)<br/>Eliminate PHI nodes for register allocation (phi-node-elimination)<br/>Two-Address instruction pass (twoaddressinstruction)<br/>Slot index numbering (slotindexes)<br/>Live Interval Analysis (liveintervals)<br/>Simple Register Coalescing (simple-register-coalescing)<br/><br /><br />Virtual Register Rewriter (virtregrewriter)<br/>Machine Copy Propagation Pass (machine-cp)<br/><br /><br /><br />Post-RA pseudo instruction expansion pass (postrapseudos) |
|                                                              |                                                              |                                                              |

仅就针对示例中的代码来说，foo函数在不同的编译选项下，代码量差别较大，跟如下一些pass相关，我们在下一节中详细介绍。

### 1.4 分析各pass的作用

对于函数foo来说，只有一个int型的入参和一个int型的返回值，因此可以用A0寄存器保存入参和返回值，而不需要借用堆栈来传递参数；同时，对于函数体来说，也只需要一个寄存器A1来保存变量的值（=10），乘法结果可以直接存放在A0寄存器中；可见，1.3节中，O1的代码，相对于O0和Odefault来说，分别有36Bytes和16Bytes的减少，但是完全可以保证程序的正确执行。

#### 1.4.1 prologepilog pass

 RISC-V 的 PrologEpilog pass 的作用是生成适用于 RISC-V 目标体系结构的函数 Prolog 和 Epilog 代码，更进一步就是生成 RISC-V 目标体系结构下函数的 Prolog 和 Epilog 代码，以确保正确地保存和恢复寄存器状态，设置和恢复堆栈指针，以及执行其他必要的初始化和清理操作。

- RISC-V 的 Prolog 包括保存调用者保存寄存器 (callee-saved registers)，设置堆栈帧，以及执行其他可能需要的初始化操作。
- RISC-V 的 Epilog 包括清理操作，例如恢复寄存器状态，恢复堆栈指针，以及执行其他可能需要的清理操作。
- 这些操作通常是由目标后端的代码生成器负责生成，它们会生成适用于 RISC-V 目标体系结构的机器代码指令序列

| 编译选项O0                                                   | 编译选项Odefault                                             | 编译选项O1                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| foo(int):<br/>        addi    sp, sp, -16<br/>        sw      ra, 12(sp)<br/>        sw      s0, 8(sp)<br/>        addi    s0, sp, 16<br/>        sw      a0, -12(s0)<br/>        lw      a0, -12(s0)<br/>        <br /><br />       lw      ra, 12(sp)<br/>        lw      s0, 8(sp)<br/>        addi    sp, sp, 16<br/>        ret | foo(int):  <br/>        addi    sp, sp, -16<br/>        <br /><br />        sw      a0, 12(sp)<br/>        mv      a0, a1<br/>        addi    sp, sp, 16<br/>        ret | foo(int):  <br/>        li      a1, 10<br/>        mul     a0, a0, a1<br/>        ret |

如上表所示，O0编译选项下的很多操作是冗余且重复的，而且是没必要的。例如，将A0寄存器的值，存入堆栈，然后从堆栈中再取出来放到A0，完全可以优化掉而不影响程序执行的正确性。而且这里访问堆栈，是通过S0寄存器来索引的，可能是为了兼容多核处理器，不同线程使用不同的S0FP指针；但是这在单核处理器中是完全不需要的。

 对于Odefault编译选项来说，尽管省去了S0寄存器作为堆栈，但是仍然在计算出返回值（存放在了A1寄存器中）以后，将A0寄存器的压入堆栈，然后再将A1寄存器的返回值复制到A0寄存器中。 

相比较来说，O1的代码，更符合逻辑，也更为简洁，且可以保证执行的正确性。如前所述，对于函数foo来说，A0寄存器可以作为入参和返回值的寄存器，无需占用堆栈空间。

#### 1.4.2 sroa pass

 在 LLVM 中，"SROA" 代表 "Scalar Replacement of Aggregates"，是一种通用的优化传递，它在不仅仅限于 RISC-V，可以应用于各种指令集体系结构。SROA 的主要作用是将聚合数据类型（如结构体或数组）拆分成单独的标量数据类型，以提高数据局部性和内存访问效率。 这是一个广为人知的聚合数据类型的标量替代转换。该转换将聚合类型（结构体或数组）的alloca指令分解为适当的情况下，为每个成员创建独立的alloca指令。然后，如果可能，它将这些独立的alloca指令转换为清晰的标量静态单赋值（SSA）形式。 

其主要目的是提高数据局部性，将聚合数据类型拆分成单独的标量数据类型可以更好地利用 CPU 的缓存，因为它允许对数据的局部访问。这可以降低内存访问的延迟，从而提高程序的性能。 

- SROA 传递分析程序中的聚合数据类型（例如结构体或数组），并查找其中的标量成员。标量成员是那些没有嵌套聚合类型的成员。
- 一旦找到这些标量成员，SROA 传递会将聚合数据类型替换为相应数量的标量数据类型。这意味着一个结构体变量可能会被替换为多个独立的变量。
- 同时，SROA 还会更新程序中的相关指令，以正确引用这些新的标量变量。
- 最终，SROA 传递的目标是在不改变程序语义的前提下，提高数据访问效率。

SROA pass比较抽象，理解起来可能有些困难，可以对比SROA pass执行前后，foo函数的IR代码差别，来理解：

| SROA执行前                                                   | SROA执行后                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| define dso_local i32 @foo(int)(i32 %var) {<br/>entry:<br/>  %var.addr = alloca i32, align 4<br/>  store i32 %var, i32* %var.addr, align 4<br/>  %0 = load i32, i32* %var.addr, align 4<br/>  %mul = mul nsw i32 %0, 10<br/>  ret i32 %mul<br/>} | define dso_local i32 @foo(int)(i32 %var) {<br/>entry:<br/>  %mul = mul nsw i32 %var, 10<br/>  ret i32 %mul<br/>} |
|                                                              |                                                              |

#### 1.4.3 GlobalOptPass pass

 在 LLVM 中，"GlobalOptPass"（全局优化传递）是一种优化传递，用于进行全局范围的优化。它的主要作用是在整个程序中执行一系列优化，以减少冗余计算、提高性能和减小生成的机器代码的大小。虽然 "GlobalOptPass" 并不是特定于 RISC-V 指令集的传递，但它可以应用于 RISC-V 架构的代码，以进行各种全局级别的优化。 

"GlobalOptPass" 通过识别和消除程序中的冗余计算，可以减少执行时的计算开销。通过各种代码转换和优化，可以提高程序的执行速度。"GlobalOptPass" 还可以减小生成的机器代码的大小，从而减少内存占用和提高缓存局部性。

- "GlobalOptPass" 在整个程序中分析和优化代码。它可以执行多种传递，包括常量折叠、死代码消除、冗余计算消除、无用代码消除等，以改进代码的质量和性能。
- 该传递可以应用于中间表示（IR），并尝试识别和消除在不同函数和模块之间共享的计算。
- "GlobalOptPass" 的原理是基于静态代码分析，它不依赖于运行时信息，因此适用于编译时优化。

在1.3小节提供的示例中，因为test2变量的计算结果，并没有被后续代码使用，属于无效代码，或者死代码（Dead-Code），因此被优化掉，使得main函数中并未调用foo函数。

而对于foo函数来说，在1.4.2小节中介绍的A0和A1间传递的部分冗余代码也被优化掉；同时，对堆栈操作的指令也被优化掉，因为foo函数在执行的时候，并不需要访问堆栈。

### 1.5 总结

编译器的优化选项和优化pass，是一项极为复杂的软件工程，这里只是针对示例程序foo函数的编译过程介绍了三个相关的pass，但是LLVM的O1编译选项，提供了更多的优化pass；而且，在整个程序的编译过程中，不同的pass，被分类为分析pass和转换pass，共同完成整个程序的编译工作。

简单的来说，LLVM O1编译选项中，常见的有下面这些：

- **Dominator Tree Construction**: 在构建支配树（Dominator Tree）时，Clang 确保正确地理解控制流结构，以便更好地进行其他优化。
- **Loop Canonicalization and Code Motion**: 此传递旨在对循环执行进行规范化，以便后续的循环变换和优化可以更好地应用。
- **Loop Data Prefetching**: 这个传递尝试在循环内引入数据预取（data prefetching）指令，以提高内存访问的性能。
- **Loop Invariant Code Motion**: 此传递检测循环内不变的代码，并将其移动到循环之外，以减少不必要的计算。
- **Loop Unrolling**: 循环展开（Loop Unrolling）尝试将循环内的迭代次数减小，以减少循环控制开销，并可能提高内存局部性。
- **Partial Redundancy Elimination (PRE)**: 部分冗余消除（PRE）是一种常量传播和代码移动的优化，旨在减少冗余计算。
- **Early CSE (Common Subexpression Elimination)**: 早期公共子表达式消除（Early CSE）传递尝试通过删除重复计算来减少不必要的计算。
- **Scalar Replacement of Aggregates (SRA)**: 这个传递试图将聚合数据类型（如结构体）中的标量成员拆分为单独的变量，以提高数据局部性。
- **Loop Unswitching**: 循环分支开关（Loop Unswitching）尝试将条件不依赖于循环迭代的部分移到循环之外，以提高性能。
- **GEP (GetElementPtr) Simplification**: GEP 简化传递尝试简化 `getelementptr` 指令，以减少冗余计算。
- **Dead Code Elimination**: 此传递检测和删除不会影响程序输出的死代码。
- **Control Flow Integrity (CFI) Checks Insertion**: CFI 是一种安全性特性，它帮助检测和防止恶意代码注入。此传递可能会插入 CFI 检查。

完整的LLVM O1优化pass，如下面所示：

```
Pass Arguments:  -tti -targetlibinfo -assumption-cache-tracker -targetpassconfig -machinemoduleinfo -tbaa -scoped-noalias-aa -profile-summary-info -collector-metadata -machine-branch-prob -domtree -basic-aa -aa -objc-arc-contract -pre-isel-intrinsic-lowering -atomic-expand -domtree -basic-aa -loops -loop-simplify -scalar-evolution -canon-freeze -iv-users -loop-reduce -basic-aa -aa -mergeicmps -loops -lazy-branch-prob -lazy-block-freq -expandmemcmp -gc-lowering -shadow-stack-gc-lowering -lower-constant-intrinsics -unreachableblockelim -loops -postdomtree -branch-prob -block-freq -consthoist -replace-with-veclib -partially-inline-libcalls -expandvp -scalarize-masked-mem-intrin -expand-reductions -loops -codegenprepare -domtree -dwarfehprepare -safe-stack -stack-protector -basic-aa -aa -loops -postdomtree -branch-prob -lazy-branch-prob -lazy-block-freq -finalize-isel -lazy-machine-block-freq -early-tailduplication -opt-phis -slotindexes -stack-coloring -localstackalloc -dead-mi-elimination -machinedomtree -machine-loops -machine-block-freq -early-machinelicm -machinedomtree -machine-block-freq -machine-cse -machinepostdomtree -machine-sink -peephole-opt -dead-mi-elimination -riscv-merge-base-offset -riscv-insert-vsetvli -detect-dead-lanes -processimpdefs -unreachable-mbb-elimination -livevars -machinedomtree -machine-loops -phi-node-elimination -twoaddressinstruction -slotindexes -liveintervals -simple-register-coalescing -rename-independent-subregs -machine-scheduler -machine-block-freq -livedebugvars -livestacks -virtregmap -liveregmatrix -edge-bundles -spill-code-placement -lazy-machine-block-freq -machine-opt-remark-emitter -greedy -virtregrewriter -stack-slot-coloring -machine-cp -machinelicm -removeredundantdebugvalues -fixup-statepoint-caller-saved -postra-machine-sink -machine-block-freq -machinedomtree -machinepostdomtree -lazy-machine-block-freq -machine-opt-remark-emitter -shrink-wrap -prologepilog -branch-folder -lazy-machine-block-freq -tailduplication -machine-cp -postrapseudos -machinedomtree -machine-loops -post-RA-sched -gc-analysis -machine-block-freq -machinepostdomtree -block-placement -fentry-insert -xray-instrumentation -patchable-function -branch-relaxation -riscv-make-compressible -funclet-layout -stackmap-liveness -livedebugvalues -riscv-expand-pseudo -riscv-expand-atomic-pseudo -lazy-machine-block-freq -machine-opt-remark-emitter
```

