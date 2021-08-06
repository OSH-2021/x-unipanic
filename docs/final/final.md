# 结题报告

- - [结题报告](#结题报告)
    * [1 项目简介](#1-项目简介)
    * [2 背景和立项依据](#2-背景和立项依据)
      + [2.1 Unikernel的介绍](#21-Unikernel的介绍)
      + [2.2 Unikernel的优势](#22-Unikernel的优势)
      + [2.3 Unikernel的问题](#23-Unikernel的问题)
      + [2.4 立项依据](#24-立项依据)
      + [2.5 前瞻性与重要性分析](#25-前瞻性与重要性分析)
    * [3 相关工作](#3-相关工作)
      + [3.1 Hermitux](#31-Hermitux)
        - [3.1.1 Linux ABI](#311-Linux-ABI)
        - [3.1.2 系统概览](#312-系统概览)
        - [3.1.3 加载时二进制可兼容性](#313-加载时二进制可兼容性)
        - [3.1.4 运行时二进制可兼容性](#314-运行时二进制可兼容性)
        - [3.1.5 重写syscall](#315-重写syscall)
      + [3.2 KylinX](#32-KylinX)
        - [3.2.1 Linux的fork](#321-Linux的fork)
        - [3.2.2 KylinX的架构](#322-KylinX的架构)
        - [3.2.3 KylinX实现fork的思路](#323-KylinX实现fork的思路)
    * [4 unipanic-改进Hermitux的思路](#4-unipanic-改进Hermitux的思路)
      + [4.1 支持fork](#41-支持fork)
      + [4.2 优化重写syscall](#42-优化重写syscall)
    * [5 unipanic实现效果和测试](#5-unipanic实现效果和测试)
      * [5.1 测试fork](#51-测试fork)
      * [5.2 测试syscall rewriter](#52-测试syscall-rewriter)
    * [6 总结](#6-总结)
    * [7 致谢](#7-致谢)
    * [参考文献](#参考文献)

## 1 项目简介

云计算场景下，Unikernel具有轻量、安全、快速的优势，但它被一些不足限制了应用范围。
本项目旨在通过提升Unikernel的二进制兼容性，来扩大它的应用范围。
我们主要参照了已有的Hermitux项目，在此基础上做出了两点改进：

1. 增加了支持fork()系统调用，从而使Unikernel支持多进/线程；
2. Hermitux项目改写了系统调用以减少上下文切换，提高速度；我们组修改了改写系统调用的逻辑，从而能改写更多的系统调用，进一步提高速度。

## 2 背景和立项依据

### 2.1 Unikernel的介绍

![unikernel_structure](https://github.com/OSH-2021/x-unipanic/blob/master/docs/final/img/unikernel_structure.png)

Unikernel 是使用 LibOS 构建的一个专门的、单一地址空间的操作系统。将操作系统的最少必要组件和程序本身打包在一起，直接运行在虚拟层或硬件上，显著提升了隔离性和效率。

目前的较为成熟的 Unikernel 项目有：

- MigrateOS，使用 OCaml 进行开发
- HaLVM，使用 Haskell
- IncludeOS，使用 C++
- Clive，使用 Go

### 2.2 Unikernel的优势

1. 性能好。内核和应用程序没有隔离，运行在同一个地址空间中，消除了用户态和内核态转换以及数据复制的开销；
2. 体积小。通过仅包含必要的运行环境和内核函数，Unikernel 的体积非常小。通过对网络功能的裁剪，HermiTux 内核在硬盘上只占用 57 kb 的空间；
3. 启动快。Unikernel 裁剪了大部分程序不需要用到的内核功能；
4. 安全。Unikernel 提供了与传统虚拟化技术（如 KVM）相等的隔离性，同时由于功能的单一化，减小了攻击面。

### 2.3 Unikernel的问题

1. 容易造成内存的浪费。为 Unikernel 分配的内存往往不会被完全利用，但无法释放回 HostKernel；
2. 没有进程。Unikernel 为单应用内核；
3. 无法调试。Unikernel 缺少调试的必要组件如 gdb；
4. 不支持二进制。在很多场景下（如无法获得源码），我们需要执行二进制文件，但这在Unikernel上是很难实现的，需要程序员理解Unikernel的底层逻辑，给程序员编写软件带来了很大麻烦。

我们组主要针对2和4进行改进。

### 2.4 立项依据

- Unikernel 以其轻量、高效和安全性，在云计算等领域颇具潜力
- 目前的 Unikernel 实现均要求对应用的重构，在实际应用中无法获取程序源码、程序依赖未被支持等问题非常常见
- 将应用打包为 Unikernel 要求大量的专业知识，步骤繁琐

我们希望在保持 Unikernel 现有优势（高效、安全、轻量）的前提下，改善 Unikernel 对二进制程序的支持，从根本上解决上述困难。

### 2.5 前瞻性与重要性分析

Unikernel 是一个安全、轻量而高效的运行环境。其应用前进十分广泛，被誉为“Linux 领域的下一代支配者”。

但目前 Unikernel 推广十分缓慢，部分原因如下：

- 如果无法获得程序源代码，重新编译和链接将无从进行，也就不可能打包到 Unikernel；
- 对二进制文件的逆向往往会因为编译过程中的剥离和混淆难以进行；
- 移植上古代码时，如果 Unikernel 不支持对应语言，就必须重写项目；
- 让 Unikernel 支持某种语言十分困难，Unikernel 通常只支持一小部分的内核特性和软件库。如果语言用到了不支持的内容，就需要重写应用，很多情况下这意味着应用完全不可能被移植；
- 编写 Unikernel 模型，会给程序员添加负担；
- Unikernel 使用复杂的构建工具，将一些传统应用的大型构建基础架构（大量的 Makefile、autotools/cmake 环境）加入 Unikernel 工具链是十分麻烦

如果可以直接将二进制程序移植到 Unikernel，上述问题都会迎刃而解。而目前致力于提供二进制兼容性的 Unikernel 项目 HermiTux 仍有较大改进空间，因此我们将改善 HermiTux 二进制兼容性作为立项目标。

## 3 相关工作

我们主要参考了Hermitux和KylinX项目，下面对这两个项目做介绍。

### 3.1 Hermitux

Hermitux主要是基于[hermitcore](https://github.com/hermitcore/rusty-hermit)这一Unikernel架构做了二进制支持，并重写syscall以保证性能。

#### 3.1.1 Linux ABI

为了提供二进制兼容性，HermiTux 的内核需要遵守构成 Linux ABI 的一组规则。这些规则可以大致分为加载时规则和运行时规则。

加载时规则包括支持的二进制格式（ELF）、应用程序可以访问64位地址空间的哪一部分。通过从二进制文件加载段来设置地址空间的方法，以及特定的寄存器状态和堆栈布局（命令行参数、环境变量、ELF辅助向量）预期在应用程序入口点。

运行时规则包括用于触发系统调用的指令，以及包含其参数和返回值的寄存器。最后，Linux 应用程序还希望通过读写各种虚拟文件系统（/proc， /sys等）来与操作系统通信，以及通过与内核共享的内存区域: vDSO/vsyscall。

#### 3.1.2 系统概览

HermiTux 的设计目标是在加载和运行时模拟 Linux ABI，同时提供 Unikernel 原则。通过使用自定义ELF加载器来确保加载时间约定。而运行时规则的遵循是通过在 HermiTux 的内核中实现一个类似 Linux 的系统调用处理程序。

HermiTux 还维护了一些单内核的优点（快速的系统调用和模块化）。

[![img/hermitux_systm.png](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/hermitux_systm.png)](https://github.com/OSH-2021/x-unipanic/blob/master/docs/research/img/hermitux_systm.png)

上图展示了系统的概览视图。

1. 在启动时，hypervisor 为客户机分配内存作为其物理内存；
2. 接下来，hypervisor 在该区域的特定位置加载 HermiTux 内核（图中的 A），然后在ELF元数据中指出的位置从Linux二进制文件加载可加载的 ELF 段；
3. 在加载过程之后，控制被传递给客户端并初始化内核。设置页表是为了构建一个同时包含内核和应用程序的地址空间。遵循单核原则，内核和应用程序都在保护环0中执行；
4. 在初始化之后，内核按照Linux的加载约定为应用程序分配和初始化一个栈，然后跳转到可执行入口点B，该入口点B的地址是在加载时从ELF元数据中读取的。在应用程序执行期间，系统调用将根据Linux惯例，使用syscall x86-64指令执行；
5. 内核通过实现一个系统调用处理程序来捕获这样的调用，该处理程序识别被调用的系统调用，从CPU寄存器中确定参数，并调用所考虑的系统调用 C 的 HermiTux 实现。

#### 3.1.3 加载时二进制可兼容性

Linux将48位虚拟地址空间的上半部分专用于内核，而由于 HermiTux 内核对虚拟/物理内存的需求非常小，因此可以将其定位在为应用程序保留的区域下面。这使应用程序可以访问虚拟地址空间的主要部分。

HermiTux 支持如下的动态编译二进制文件：当加载器检测到这样的二进制文件时，它加载并将控制传递给动态加载器，轮到后者来加载应用程序和库依赖项，并负责符号的重定位。

与 KylinX 相反，HermiTux 不在多个 Unikernel 之间共享虚拟地址空间中的动态库。这主要是出于安全原因，因为虚拟机之间的共享内存可能使它们容易受到侧通道的攻击（例如 Flush + Reload 或 Prime + Probe）。

HermiTux的目标是同时支持静态和动态的链接。

#### 3.1.4 运行时二进制可兼容性

HermiTux支持Linux规定的系统调用，而现有的Unikernels内核不支持。HermiTux实现了在执行syscall指令时调用的系统调用处理程序。在系统调用级别上与应用程序接口是HermiTux提供的二进制兼容性的核心。

但是，这并不代表Linux系统调用接口的完全重新实现：虽然它相当大（超过350个系统调用），但应用程序通常只使用该接口的一小部分。只要实现200个系统调用，就可以支持90%的标准发行版二进制文件。

HermiTux 的原型实现了83个系统调用。

#### 3.1.5 重写syscall

由于系统调用syscall依赖于上下文切换，十分耗时，Hermitux通过重写syscall来减少程序运行的时间，延续Unikernel的优势。

HermiTux依靠两种技术提供快速的系统调用（Fastcall）。

对于静态二进制程序，可使用二进制工具将应用程序中发现的syscall指令用常规函数调用重写到相应的单核实现中。

对于动态链接的程序，有以下特点：

1. 大部分的系统调用都是由C库完成的；
2. C库接口定义得非常好，所以可以在运行时链接一个程序，而这个程序的共享C库并不是程序编译时的库。

然后，对动态链接的程序使用HermiTux设计的一个单核感知的C库，它是直接对内核进行函数调用，代替系统调用。这种技术称为库替换。

由于要实现二进制兼容性，HermiTux 中的系统调用代码库相对较大。

设计内核时，每个系统调用的实现都可以在内核构建时编译进去或编译出来。

除了减少内存占用外，这样做还有比传统的系统调用过滤（如seccomp）更强的安全优势：不仅无法调用相关的系统调用，而且它们的实现完全不在内核中，这意味着它们不能被用于代码重用攻击。

HermiTux设计了一个二进制分析工具，能够扫描一个可执行文件，并检测该程序可以进行的各种系统调用。

HermiTux的内核中实现了一个基本的RAM文件系统——MiniFS，从而在这方面消除了对主机的依赖。

![hermitux_rewritesyscall](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/hermitux_rewritesyscall.png)

Hermitux原本的syscall如上图所示，从syscall这条指令开始，向后寻找指令，找到若干条指令和syscall组合起来成为5字节及以上（x86是CISC指令集，使用不定长指令，syscall占2字节，jmp占5字节，我们要使用jmp的话，需要其他指令和syscall拼凑够5字节）。

这样，就可以把原本的syscall和与它拼凑的指令一起替换成jmp和若干条空指令。

使用jmp指令，可以跳转到内核中一段空闲的空间，并在这段空间里填入：被改写的syscall指令（从系统调用改写为函数调用），和syscall打包被替换的指令，返回指令。

### 3.2 KylinX

#### 3.2.1 Linux的fork

![linux_fork](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/linux_fork.png)

Linux的fork如上图所示，但是传统的Unikernel是不支持fork的，也就不支持多进程；我们参考的Hermitux项目也不支持fork，所以我们决定让我们的unipanic支持fork。

这里我们参考了KylinX的实现，让Unikernel支持fork()这一系统调用，从而支持进程。

#### 3.2.2 KylinX的架构

首先，KylinX基于[ Xen Project ](https://xenproject.org/users/why-xen/)的Unikernel架构。Xen的hypervisor运行两类虚拟机，一类叫Dom0，一类叫DomU。Dom0具有最高权限，可以管理DomU。

![kylinx_structure](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/kylinx_structure.png)

KylinX的架构如上图所示。

#### 3.2.3 KylinX实现fork的思路

KylinX的思路是：将hypervisor视为OS，将运行在hypervisor上的虚拟机VM视为进程(process)，每当一个进程要fork出一个新进程时，就让hypervisor启动一个新的虚拟机，然后将hypervisor调动Dom0管理新启动的DomU。

![kylinx_fork_structure](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/kylinx_fork_structure.png)

![kylinx_fork_process](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/kylinx_fork_process.png)

上图显示了Dom0处理父进程(Parent DomU)fork出子进程(Child DomU)的请求。

## 4 unipanic-改进Hermitux的思路

### 4.1 支持fork

我们参照了KylinX实现fork的方式，也想让hypervisor启动新的虚拟机作为子进程。

因为我们用的Hermitux基于hermitcore架构，KylinX基于Xen架构，两者并不完全一致；我们在阅读hermitcore的代码时，发现它是用`mmap()`生成一块虚拟地址空间，并在这块空间上创建虚拟机；我们尝试过用`memcpy()`来复制`mmap()`生成的这块虚拟地址空间，也的确有效（看起来是安全的）。

```C++
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
int main()
{
    int size = 200 * 1024 * 1024;
    void *run = mmap(0, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    void *run2 = mmap(0, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    *(int*)run = 10;
    int a ;
    scanf("%d",&a);
    memcpy(run2, run, size);
    printf("%d\n", *(int*)run);
    printf("%d\n", *(int*)run2);

    scanf("%d",&a);

    return 0;
}
```

但是Hermitux的代码风格有点混乱，（比如很多处出现线程共用不该共用的地方，全局变量满天飞，有些地方还有bug x），我们就没有尝试在Hermitux的代码上修改创建虚拟机的部分，而是通过复制hypervisor的方式来实现fork，这样改动比较容易，且不易出错。

这种实现方式显然性能较低，会让整个系统显得很笨重，但我们这里只是做出了对于Unikernel支持fork的一种可能的尝试，希望后面会有更好的实现fork的思路。

![unipanic_fork_structure](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/unipanic_fork_structure.png)

上图便是我们实现fork的架构。

![unipanic_fork_process](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/unipanic_fork_process.png)

具体步骤如上图所示。运行子程序时，Hermitux在检测到syscall后会跳入syscall处理程序`syscall_handler()`，我们就在`syscall_handler()`里加入了对fork的判断，如果系统调用是fork，就会跳入我们写的fork处理程序里。

在我们的fork处理程序里，先保存CPU状态，调用`fork()`拿到进程序号pid，接着判断是父进程还是子进程：若是父进程，就什么也不做，直接返回；若是子进程，就拷贝父进程的hypervisor的内存、拷贝父进程的CPU状态，然后返回，子进程自然就会接着父进程的工作继续做了。

### 4.2 优化重写syscall

若使用Hermitux重写syscall的函数，可以把Glib中679个syscall重写545个，重写效率为78.2%。

我们觉得Hermitux重写syscall的效率太低了，因为从syscall开始向后查找会有几类不能重写syscall的情况：

* syscall后面的基本块（basic block）太小或没有基本块，不足以与syscall组合打包并被替换；
* syscall所在的基本块不是后一基本块的唯一来源（有别的指令跳转到后一基本块）；
* syscall后面的基本块使用了rip或fs寄存器；
* syscall后面的基本块包含不能被编译的语句

这些不能重写的syscall会被Hermitux排除掉，就剩下78.2%的syscall可以被重写了。

我们观察到了以下情况：

* 调用syscall之前，通常会有指令`mov *ax`设置eax、rax等以ax为后缀的寄存器的值，作为系统调用号，内核再根据系统调用号执行系统调用；
* `move *ax`作为跳转目的地的可能性极小。我们可以安全地把设置这类寄存器的语句与syscall一起打包（即从向后打包转变为向前打包），再做替换，这样可以避免一些不能重写syscall的情况；
* `move *ax`占用5字节，足以和syscall一起打包并被替换了

![unipanic_rewritesyscall](https://github.com/OSH-2021/x-unipanic/raw/master/docs/final/img/unipanic_rewritesyscall.png)

unipanic向前打包指令如上图所示。

我们在被Hermitux筛掉的syscall中再做分析，如果只分析到syscall前一条语令，可以比Hermitux多重写90个syscall，重写效率为91.1%；

若分析syscall的前两条指令，可将679个syscall全部重写，重写效率为100%。

## 5 unipanic实现效果和测试

### 5.1 测试fork

我们的测试代码为：

```c++
#include <stdio.h>
#include <unistd.h>

int main()
{
    int ret = fork();

    if (ret != 0)
    {
        printf("prog: Parent, pid = %d\n", ret);
    }
    else
    {
        printf("prog: Child\n");
    }
}
```

测试结果为：

```bash
gcc 3.c -o 3.out -static
sudo HERMIT_ISLE=uhyve HERMIT_TUX=1 HERMIT_VERBOSE=1 HERMIT_DEBUG=0 \
	../build/local_prefix/home/elsa/Code/hermitux-kernel/prefix/bin/proxy 
...
prog: Parent, pid = 2781
uhyve_atexit prog_cnt = 0
...
prog: Child
uhyve_atexit prog_cnt = 1

uhyve_atexit Dump kernel log:
================
...
```

证明我们对fork的实现是成功的。

### 5.2 测试syscall rewriter

我们的测试代码如下：

```c++
#include <stdio.h>
#include <stdlib.h>
int main()
{
    char buf1[40];
    FILE *f1 = fopen("t1", "r");
    FILE *f2 = fopen("t2", "r");
    FILE *f3 = fopen("t3", "w");
    fgets(buf1, 20, f1);
    fgets(buf1 + 15, 20, f2);
    fputs(buf1, f3);
    fclose(f1);
    fclose(f2);
    fclose(f3);
}
```

测试结果如下（`test.asm`和`TEST_FAST.ASM`分别是重写syscall前和重写后的测试代码的反汇编指令。这里截取部分比较结果）：

```bash
...
***** test.asm
  48d5ca:       be 81 00 00 00          mov	 $0x81,%esi
  48d5cf:       b8 ca 00 00 00          mov  $0xca,%eax
  48d5d4:       0f 05                   syscall
  48d5d6:       48 83 7c 24 18 00       cmpq $0x0,0x18(%rsp)
***** TEST_FAST.ASM
  48d5ca:       be 81 00 00 00          mov  $0x81,%esi
  48d5cf:       e9 a3 4c da ff          jmpq 
  232277 <catch_hook+0x23221f>
  48d5d4:       90                      nop  
  48d5d5:       90                      nop  
  48d5d6:       48 83 7c 24 18 00       cmpq $0x0,0x18(%rsp)
*****
...
```

比较我们重写syscall之前和重写之后的反汇编指令，syscall指令已被替换为jmpq指令，说明重写成功。

## 6 总结

我们组在改进Unikernel的道路上给出并测试了一个的思路：

* 可以实现Unikernel的二进制支持，让它能支持无源码的情况；
* 让Unikernel支持fork这一系统调用，从而支持多进/线程的场景

我们主要参照了KylinX和Hermitux这两个项目，KylinX项目帮助我们得到了实现fork的思路，Hermitux主要实现Hnikernel的二进制支持，并且开源，我们在Hermitux的源码上进行改动：

* 将一个hypervisor及其虚拟机作为一个进程，通过复制hypervisor的内存及CPU状态达到创建子进程并共享内存空间的目的；
* 将syscall这条指令和它之前的指令一起打包替换，而不是向后打包（这样重写syscall的范围更广），从而将系统调用替换为函数调用，并保证原来被打包替换的语句照常执行，减少系统调用的上下文切换，保持Unikernel因没有系统调用而具有的优良运行速度

希望通过我们的努力，既能改进Unikernel的不足，又能让它保持原来的优点，在云场景下得到更多应用。

## 7 致谢

感谢全体小组成员的付出，感谢邢老师为我们组选题提供了宝贵的思路和意见，感谢Hermiutux原作者给我们提出的建议和支持。

## 参考文献

- [1] 高策. 2017. Unikernel：从不入门到入门. [https://gaocegege.com/Blog/%E5%AE%89%E5%88%A9/unikernel-book](https://gaocegege.com/Blog/安利/unikernel-book)
- [2] Bryan Cantrill. 2016. Unikernels are unfit for production. https://www.joyent.com/blog/unikernels-are-unfit-for-production
- [3] Hacker News 2017. 2016. Unikernels are entirely undebuggable. https://news.ycombinator.com/item?id=10954132
- [4] Pierre Olivier, Daniel Chiba, Stefan Lankes, Changwoo Min, and Binoy Ravindran. 2019. A binary-compatible unikernel. In Proceedings of the 15th ACM SIGPLAN/SIGOPS International Conference on Virtual Execution Environments (VEE 2019). Association for Computing Machinery, New York, NY, USA, 59–73. DOI:https://doi.org/10.1145/3313808.3313817
- [5] Filipe Manco, Costin Lupu, Florian Schmidt, Jose Mendes, Simon Kuenzer, Sumit Sati, Kenichi Yasukata, Costin Raiciu, and Felipe Huici. 2017. My VM is Lighter (and Safer) Than Your Container. In Proceedings of the 26th Symposium on Operating Systems Principles (SOSP ’17). ACM, New York, NY, USA, 218–233. DOI:http://dx.doi.org/10.1145/3132747.3132763
- [6] OSv Contributors 2014. Porting native applications to OSv: problems you may run into. (2014). https://github.com/cloudius-systems/osv/wiki/Porting-native-applications-to-OSv.
- [7] Sören Bleikertz. 2011. How to run Redis natively on Xen. https://openfoo.org/blog/redis-native-xen.html.
- [8] Stefan Lankes, Simon Pickartz, and Jens Breitbart. 2016. HermitCore: a unikernel for extreme scale computing. In Proceedings of the 6th International Workshop on Runtime and Operating Systems for upercomputers (ROSS 2016). ACM
- [9] Unikernel Blog. 2017. Unikernels are Secure. http://unikernel.org/blog/2017/unikernels-are-secure.
- [10] Florian Schmidt. 2017. uniprof: A Unikernel Stack Profiler. In Proceedings of the ACM Special Interest Group on Data Communication Conference (Posters and Demos) (SIGCOMM’17). ACM, 31–33.
- [11] Pierre Olivier, Daniel Chiba, Stefan Lankes, Changwoo Min, and Binoy Ravindran. 2019. A binary-compatible unikernel. In Proceedings of the 15th ACM SIGPLAN/SIGOPS International Conference on Virtual Execution Environments (VEE 2019). Association for Computing Machinery, New York, NY, USA, 59–73. DOI:https://doi.org/10.1145/3313808.3313817
- [12] Yiming Zhang, Jon Crowcroft, Dongsheng Li and Chengfen Zhang, Huiba Li,  Yaozheng Wang and Kai Yu, Yongqiang Xiong, Guihai Chen. 2018. In the Proceedings of the 2018 USENIX Annual Technical Conference (USENIX ATC ’18).   [KylinX: A Dynamic Library Operating System for Simplified and Efficient Cloud Virtualization | USENIX](https://www.usenix.org/conference/atc18/presentation/zhang-yiming)