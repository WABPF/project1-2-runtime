---
title: Faasm / WASM serverless
date: 2023-02-06 11:50:33
---

论文地址：https://arxiv.org/abs/2002.09344
论文翻译：https://zhuanlan.zhihu.com/p/134152415


## 如何基于WASM设计一个serverless平台

由于WASM具有内存安全的特性，使用WASM实现轻量级的隔离，以取得较高的性能。

1. 将用户function编译成WebAssembly，采用WASM runtime（例如WAVM）对function进行执行。
2. 执行操作由“调度器”进行（Faasm），“调度器”使用一个“执行线程”进行任务执行，并使用CGroup对“执行线程”进行约束。
3. 为了实现完整的serverless平台，例如io调用、状态共享，需要加入相应的API支持（Faaalets提供了相应的API——host interface）。

## Faaslets/Faasm总结

### 1）Faaslets

Faaslets是一种software-based isolation。这是一种用于高性能无服务计算的隔离抽象。Faaslets使用WebAssembly提供的软件故障隔离（SFI，*software-fault isolation*, provided by *WebAssembly*）来隔离已执行函数的内存，并允许同一地址空间的函数之间共享内存区域。

Faaslets具有以下几个特性：

1. [隔离](#is)：每个fasslet由一个运行process进行执行，利用CGroup进行线程隔离
2. 为实现高效的通信，1）提供本地[共享内存机制](#memy)（由WASM的特性保证内存安全，[WASM内存模型](https://zhuanlan.zhihu.com/p/386849387)）。2）同时提供由distributed key-value store (KVS)支持的全局状态共享。
3. 实现了[host interface](#inter)，提供API（类[POSIX](https://zh.wikipedia.org/wiki/可移植操作系统接口)的子集，基于WASI），函数可以调用API以使用各种系统功能（相当于在函数和sys之间加了一层抽象层，实现了低水平的虚拟化）。使用message bus与父进程通信，接收函数调用、共享、调用和等待其他函数等信息

![截屏2023-02-06 12.58.20](https://raw.githubusercontent.com/muchengl/pic_storage/main/uPic/%E6%88%AA%E5%B1%8F2023-02-06%2012.58.20.png)

### 2）Faasm

Faasm是一个负责调度Faaslets的runtime。Faasm可以管理多个Faaslets。Faasm通过Faaslets的状态共享机制进行调度。

如下图，ABC三个函数调用事件，Faasm instance1具有实例A，因此直接执行。Faasm instance1 缺少BC实例，为了避免冷启动，则将BC任务共享到Faasm instance2.

![截屏2023-02-06 14.19.51](https://raw.githubusercontent.com/muchengl/pic_storage/main/uPic/%E6%88%AA%E5%B1%8F2023-02-06%2014.19.51.png)

此外，Faasm使用LLVM编译function，用WAVM进行执行。

>编译过程：
>(1) 用户调用Faaslet工具链将函数编译成WebAssembly二进制文件，链接到Faaslet host interface的特定语言声明;
>(2) 生成一个由WebAssembly创建的“object file with machine code”
>(3) host interface与machine code链接以生成Faaslet可执行文件（Faaslet executable）

## Faaslets的特性

Faaslets是一种software-based isolation。这是一种用于高性能无服务器计算的新的隔离抽象。Faaslets使用WebAssembly提供的软件故障隔离（SFI，*software-fault isolation* (SFI), provided by *WebAssembly*）来隔离已执行函数的内存，同时允许同一地址空间的函数之间共享内存区域。

 FAASM是一个基于Faaslets的runtime，使用标准的Linux cgroups隔离其他资源，如CPU和网络。并为网络、文件系统访问和动态加载提供一个低级别的POSIX主机接口

> 现在的容器/vm，启动慢，内存占用大

无服务器计算可以通过一种新的isolation abstraction，以更好地支持数据密集型应用程序：

(i) 在函数之间，提供强的**内存和资源隔离**，保证安全性
(ii) 支持高效的**状态共享**。数据应该与功能共存（co-located），并直接获取，尽量减少数据运输
(iii) 允许在多个主机上扩展状态
(iv) 低内存消耗，允许在一台机器上有许多实例；
(v) 快速的实例化
(vi) 支持多种编程语言

为了实现以上特性，Faaslets具有以下特性：

1. **Faaslets轻量级的隔离**
    Faaslet依赖于软件故障隔离（SFI），它将函数限制在对其自身内存的访问。一个与Faaslet函数，连同其库和语言运行时的依赖，被编译成WebAssembly。FAASM运行时在一个地址空间内执行多个Faaslet，每个都有一个专用线程。为了实现资源隔离，每个线程的CPU周期使用Linux cgroups进行约束，网络访问使用“*network namespaces*”和“*traffic shaping*”进行限制。

2. **faaslet 高效本地/全局状态访问**
    由于faaslet共享相同的地址空间，它们可以有效地使用本地状态访问共享内存区域。这允许数据和函数的共同定位，并避免串行化的开销。faaslet使用两层状态架构，本地层提供内存共享，全局层支持跨主机分布式访问状态。
    FAASM运行时为Faaslets提供了一个状态管理API，可以对两层的状态进行细粒度控制。
3. **faaslet快速初始化**
    为了减少Faaslet第一次执行时的冷启动时间，Faaslet从suspended state启动
    suspended state：FAASM预先初始化一个Faaslet，并对其内存进行快照以获得一个原始Faaslet。proto -Faaslet用于快速创建新的Faaslet实例，可以避免初始化语言运行库的时间。理论上proto - faaslet支持跨主机恢复，并且与操作系统无关。
    （这一块类似MITOSIS，是直接使用了从内存恢复运行状态）
4. **Faaslets 主机接口**
    Faaslets通过类似POSIX-like calls与主机环境进行交互。包括网络、文件I/O、全局状态访问和库加载/链接。主机接口提供了足够的虚拟化以确保隔离，同时增加的开销可以忽略不计。

FAASM runtime使用LLVM编译器工具链将应用程序转换为WebAssembly，这一方法支持用多种编程语言编写的函数。可现有的无服务器平台集成，例如Knative。

## Faaslets High level design

![截屏2023-02-06 12.58.20](https://raw.githubusercontent.com/muchengl/pic_storage/main/uPic/%E6%88%AA%E5%B1%8F2023-02-06%2012.58.20.png)

1. <a id="memy">内存设计</a>：函数被编译为WebAssembly，被放置在它自己的私有连续内存区域中。同时Faaslets也支持共享内存区域，这允许Faaslet在WebAssembly的内存安全保证的约束下访问共享的内存状态(当需要内存共享时，Faaslet扩展WASM的字节数组，但将新页面映射到公共进程内存的指定区域（mmap）。然后可以给函数一个指向字节数组新区域的指针，但所有访问都在共享区域上执行)。
    ![截屏2023-02-06 15.22.09](https://raw.githubusercontent.com/muchengl/pic_storage/main/uPic/%E6%88%AA%E5%B1%8F2023-02-06%2015.22.09.png)

    

2. <a id="is">隔离</a>：faaslet确保了公平的资源访问。对于CPU隔离，他们使用“Linux cgroups的CPU子集”。每个函数都由共享运行时进程的专用线程执行，这个线程被分配了一个cgroup。

3. IO接口：Faaslets通过网络命名空间、虚拟网络接口和流量整形实现安全公平的网络访问。每个Faaslet在单独的名称空间中都有自己的网络接口，使用iptables规则进行配置。为了确保共享租户之间的公平性，每个Faaslet使用tc在其虚拟网络接口上应用流量整形，从而强制执行入口和出口流量速率限制。

4. <a id="inter">Faaslet host interface</a>：Faaslet host interface是一个virtualisation layer，提供了一系列API，以隔离函数于外界环境。这些借口基于WASI（这个设计有点类似gVisor，但是gVisor是模拟了整个内核）

![截屏2023-02-06 13.15.25](https://raw.githubusercontent.com/muchengl/pic_storage/main/uPic/%E6%88%AA%E5%B1%8F2023-02-06%2013.15.25.png)

5. faaslet使用message bus与父进程通信，接收函数调用，共享工作，调用和等待其他函数等信息。

## Faaslets状态共享

Faaslets提供了两层架构的状态共享机制，这一机制可帮助实现Faasm的调度。

本地层：提供对同一主机上的状态的共享内存访问;全局层：允许Faaslets在主机间同步状态。DDOs隐藏了两层状态架构，提供了对分布式数据的透明父访问



## ~~WASM是什么~~

WebAssembly为具有LLVM前端的语言提供了成熟的支持，   ![img](https://pic1.zhimg.com/80/v2-e93d22914f2939017db6c6ff463cc2d8_1440w.webp)

- 用clang编译成`.o`

- 用wasm-ld连接成`.wasm`

```sh
clang -c -O3 --target=wasm32 test.cpp
wasm-ld --no-entry --strip-all --allow-undefined --export-all test.o -o test.wasm    
```

之后就可以使用例如WAVM的runtime进行执行。
