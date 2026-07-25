# Chromium 构建工具的演进 -- 从 Ninja 到 Siso

## 目录
<details>
<summary>展开目录</summary>
  
- [TL;DR](#tldr)
  - [Chromium 构建工具演进](#chromium-构建工具演进)
  - [Google 构建工具生态](#google-构建工具生态)
- [一 为什么需要 Siso？](#一-为什么需要-siso)
  - [1.1 旧架构的性能瓶颈](#11-旧架构的性能瓶颈)
  - [1.2 Siso 的分布式编译工作流](#12-siso-的分布式编译工作流)
- [二 Siso 与 Bazel 的关系](#二-siso-与-bazel-的关系)
  - [2.1 命名背后的技术黑话：致敬 Bazel 的“香草梗”](#21-命名背后的技术黑话致敬-bazel-的香草梗)
  - [2.2 Siso 的定位](#22-siso-的定位)
- [三 Siso 替代 Ninja 细则：所有人都得换吗？](#三-siso-替代-ninja-细则所有人都得换吗)
  - [3.1 哪些场景下必须用 Siso 替代 Ninja？](#31-哪些场景下必须用-siso-替代-ninja)
  - [3.2 Ninja 这个开源项目以后不再维护了吗？](#32-ninja-这个开源项目以后不再维护了吗)
- [四 进阶多技术栈选型：现有项目该用 Siso 还是 Bazel？](#四-进阶多技术栈选型现有项目该用-siso-还是-bazel)
  - [4.1 为什么这类项目无法选择 Siso？](#41-为什么这类项目无法选择-siso)
  - [4.2 为什么 Bazel 是完美契合的解法？](#42-为什么-bazel-是完美契合的解法)
  - [4.3 构建工具直观选型矩阵](#43-构建工具直观选型矩阵)
  - [4.4 避坑提示：Bazel 的“终极代价”与轻量级替代方案](#44-避坑提示bazel-的终极代价与轻量级替代方案)
- [五 总结](#五-总结)
- [六 参考](#六-参考)
</details>

## TL;DR

* Siso 不是 Bazel 的替代品，而是 Ninja 的替代品。
* Siso 面向 Chromium / Android AOSP 构建体系。
* Bazel 面向跨语言 Monorepo。
* Siso 深度复用了 Bazel Remote Execution（REAPI）生态，但并非基于 Bazel 开发。

### Chromium 构建工具演进

| 阶段 | 说明 |
| -------- | -------------- |
| Make     | Chromium 早期使用  |
| Ninja    | 长期使用的执行器       |
| Goma     | 第一代远程编译方案（已淘汰） |
| Reclient | 第二代远程编译代理      |
| Siso     | 当前官方默认执行器      |

### Google 构建工具生态

```text
    Google Build Ecosystem

   +----------------------+
   |       Bazel          |
   |    Build System      |
   +----------+-----------+
              |
  REAPI / CAS / Remote Cache
              |
   +----------+-----------+
   |        Siso          |
   |    Build Executor    |
   +----------+-----------+
              |
        .ninja Graph
              |
             GN
                   
```
| 工具           | 定位                    | 是否属于 Chromium 当前流程 |
| ------------ | --------------------- | ------------------ |
| GN           | Build Graph Generator | ✔                  |
| Siso         | Build Executor        | ✔                  |
| Remote Cache | Artifact Cache        | ✔                  |
| RBE          | Remote Executor       | ✔                  |
| Bazel        | Build System          | ✘（仅共享 REAPI 生态）    |

---

## 一 为什么需要 Siso？

在现代超大型软件工程中，构建速度（Build Time）直接决定了开发者的生产力。以 Google 开源的 Chromium 项目为例，其代码库包含数万个源文件，若纯粹依赖单机编译，往往需要数小时甚至十几个小时。

为了攻克这一瓶颈，Google 曾长期依赖 **Ninja 构建系统** 配合远程编译代理。然而，随着项目规模的进一步膨胀，旧架构的性能触及了天花板。为此，Google Chrome Build Infra 团队开发了下一代构建工具 —— **Siso**。根据 Chromium 官方 Siso 概述，Siso 目前已全面接管 Chromium 全球开发生态。

本文将介绍 Siso 的诞生背景、命名趣闻、核心技术特性，以及它在多技术栈工程选型中与 Bazel、Ninja 的关系。

### 1.1 旧架构的性能瓶颈

在 Siso 问世之前，Chromium 的分布式编译严重依赖 **Ninja + Reclient**（或更早期的 Goma）的组合。

在这个体系中，Ninja 扮演着本地执行器的角色。当需要分布式高并发编译时，团队必须在 Ninja 之外包裹一层远程执行代理（如 Reclient）。这种架构在面对百万行级别的代码库时，暴露出了明显的性能弊端：

* **I/O 负载较高**：Ninja 作为一个纯本地构建工具，在运行期间需要频繁进行磁盘状态检查（`stat`）和本地文件 I/O。在大规模集群并发时，这种高频的磁盘交互会成为效能瓶颈。
* **架构脱节，开销增大**：由于远程执行（RBE）功能是通过外部代理（Proxy）实现的，本地进程与外部代理之间、代理与云端之间存在多层转发，增加了调用开销，也提高了整体系统的维护复杂度。

为了改变这一过时的代理架构，减少构建过程中的 I/O 损耗，Siso 应运而生。

### 1.2 Siso 的分布式编译工作流

根据 Siso 核心设计文档，在全面转向 Siso 后，Chromium 体系的现代分布式编译工作流演进得更为高效：

```text
[ GN 工具 ] ➔ 解析代码结构，生成 .ninja 依赖图描述文件
      ↓
[ Siso 执行器 ] ➔ 读取描述，并在本地执行以下高性能流水线：
      │
      ├─ 1. 计算哈希值：为每个编译单元（Action）计算全局唯一的输入 Hash 码
      ├─ 2. 检查远端缓存：比对 RBE 缓存，若命中则直接下载编译产物（无需编译）
      └─ 3. 远端高效分发（若未命中缓存）：
               │
               └─➔ [ 远端高并发 RBE 集群 ]
                        │ 包含成百上千个 Worker 节点
                        │ 并发执行编译动作（完全在内存与高速网络中交互）
                        ↓
[ 吐出最终产物 ] ➔ Siso 本地接收远端回传，完成最终组装
```

通过这套工作流，Siso 实现了两项核心技术突破：

* **减少 I/O 损耗**：Siso 内部通过单进程内存空间共享，将大量的编译状态和文件信息维护在内存中，在任务调度时尽可能避免了本地磁盘的 `stat` 和网络 I/O 开销。
* **更严格的依赖图校验**：根据官方的 Siso 与 Ninja 差异对比描述，Siso 对构建目录实施了更安全的锁定机制，执行比 Ninja 更严苛的输入/输出存在性检查，以此确保多机分布式缓存的绝对准确，避免了由于本地文件残留导致的脏编译与并发竞争问题。

---

## 二 Siso 与 Bazel 的关系

### 2.1 命名背后的技术黑话：致敬 Bazel 的“香草梗”

开发团队在命名时埋下了一个趣味的技术暗号：

* **Bazel**（Google 的旗舰级分布式构建系统）：其名字 Bazel 的发音，与英文中的常用烹饪香草 **“Basil（罗勒 / 九层塔）”** 完全相同。
* **Siso**（Google 的新一代 Chromium 构建工具）：其名字 Siso 的发音，对应日语中的 **“紫苏（Shiso）”**。

在植物学中，“紫苏”与“罗勒”同属于唇形科植物，它们是近亲，长相相似，且都是厨房里常见的调味香草。

因此，Google 团队将这个新工具命名为 Siso，是以一种幽默的技术黑话向老大哥 Bazel（罗勒）致敬，暗示 Siso 在底层架构和分布式设计思想上，与 Bazel 是一脉相承的。为了让开发者每天在终端敲击命令时更方便，团队还刻意将拼写精简为了更易输入的四个字母 `siso`。

### 2.2 Siso 的定位

Siso 的诞生本质上体现的是**工程实用主义（Pragmatism）**，而非对现有构建体系的彻底重构。它并不是因为“Siso 比 Bazel 的架构更先进”而被设计出来，而是因为保留 `GN -> .ninja` 构建链，能够以最小代价获得 Bazel Remote Execution（RBE）的能力。

对于 Chromium 而言，其拥有数百万行代码和数万个构建目标，多年来积累了庞大的 `BUILD.gn` 规则。将整个工程迁移到 Bazel 意味着必须重写规则并重新验证，迁移成本与风险极高。此外，部分轻量级构建环境（如 ChromeOS、嵌入式环境）也需要避免引入复杂的 Java（JVM）运行时依赖。

因此，Google Chrome Infra 团队选择了一条成本最低、收益最高的技术路线：

> **保持 GN 生成 `.ninja` 的流程完全不变，仅替换底层执行器 -- Ninja，使其能够原生接入 Bazel Remote Execution（REAPI）和 Remote Cache。**

从架构上看，Siso 采用了 **旁路增强（Sidecar Enhancement）** 的思路，相当于一座连接两种生态的桥梁：

* 向上：兼容 GN 与 .ninja，无须重写成熟的既有构建规则；
* 中间：采用 Go 实现全新的执行引擎，替代传统的 `Ninja` + `Reclient` 组合，显著降低内存与磁盘 I/O 开销；
* 向下：直连 Bazel REAPI，复用成熟的远程编译与分布式缓存能力。

从定位来看，Siso 并不是一个面向通用软件工程的新型构建系统，而是专门服务于 **Chromium 体系的高性能构建执行器（Build Executor）**。它的存在，本质上源于 Chromium 长期积累的工程资产与迁移约束——它并非 Bazel 的竞争者，而是 Chromium 在无法整体迁移至 Bazel 的前提下，对 Bazel Remote Execution 能力进行的一次高效率工程嫁接。

---

## 三 Siso 替代 Ninja 细则：所有人都得换吗？

### 3.1 哪些场景下必须用 Siso 替代 Ninja？

在编译 Chromium、Android AOSP 以及基于 Chromium 基础设施的所有开源项目场景中，Siso 已全面取代 Ninja。

* **对 Google 内部**：其内部的 CI/CQ 自动化流水线和员工开发机已全量切换为 Siso。
* **对外部贡献者与生态**：Chromium 官方的构建脚本中已不再对 Ninja 提供官方支持（Unsupported），原有的远程编译代理 Reclient 也被完全移除。
* **配置方式**：开发者可以通过在 `args.gn` 中显式配置 `use_siso=true`（或清除旧缓存后直接使用 `autoninja`）来让系统自动调用 Siso。若遇到兼容性故障，可临时通过 `use_siso=false` 回滚。

### 3.2 Ninja 这个开源项目以后不再维护了吗？

**不是。** Ninja 作为一个全局广泛使用的独立开源项目，依然会由社区继续维护。

Ninja 是整个软件工程界的底层基石之一，也是 CMake、Meson 等现代构建系统的默认后端。Chromium 团队宣布停止支持 Ninja，仅仅意味着 Chromium 这个单项工程的构建脚本不会再对 Ninja 进行兼容性适配和排错，并不影响 Ninja 在其他非 Chromium 项目中的正常使用。

---
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->


## 四 进阶多技术栈选型：现有项目该用 Siso 还是 Bazel？

如果您当前正面临团队的技术选型，且团队的工程技术栈涵盖了 Golang、Android、iOS、Next.js 这四类典型技术，那么在 Siso 与 Bazel 之间，毫无疑问应该选择 **Bazel**，而绝对不应该选择 Siso。

### 4.1 为什么这类项目无法选择 Siso？

Siso 无法在通用开发中替代 Bazel，这是由其原生定位决定的：

* **缺乏通用语言规则库**：Siso 没有面向全球开源社区的通用语言插件生态（如解析 Go 模块、编译 Swift 或打包 JavaScript 依赖）。
* **自建脚手架成本极高**：Siso 的底层输入只认类似 Ninja 的编译图描述文件。如果要在 Go 或 Next.js 项目中强行引入 Siso，团队必须先自行编写一个脚手架，把 Go、Node.js 的编译步骤全部翻译成 `.ninja` 文件再喂给 Siso，这在工程上属于典型的“重新发明轮子”。

### 4.2 为什么 Bazel 是完美契合的解法？

Bazel 是目前整个软件工程界少有的、能够将多技术栈装进同一个 Monorepo（代码巨仓）并进行全链路统一依赖调度的构建工具。它依托庞大的全球开源贡献者生态，提供了极其成熟的规则集支持：

* **Golang 支持**：由 Bazel 社区官方维护的 `rules_go` 规则集是目前全行业公认最强大的 Go 语言多平台交叉编译与测试规则集，完美接管 `go.mod` 依赖并提供极致的增量缓存。
* **Android / iOS 移动端双端通吃**：通过 `rules_apple` 套件和官方移动端编译工具链，开发者可以用一套工具一键编译出 iOS 的 `.ipa` 和 Android 的 `.apk`/`.aab` 产物。
* **Next.js 与前端生态优化**：通过成熟的 `rules_js` 构建生态，Bazel 可以基于 pnpm 等现代包管理器完美虚拟出高效的 `node_modules` 树，在大型微前端集群中，能让前端的 CI 部署速度通过分布式缓存提升数倍。

### 4.3 构建工具直观选型矩阵

| 特性 / 维度 | 🌿 Siso (紫苏) | 🌿 Bazel (罗勒) |
| :--- | :--- | :--- |
| **核心定位** | Ninja 描述文件的分布式高并发执行器 | 跨语言、跨平台的通用构建框架 |
| **Golang 支持** | ❌ 无原生规则，需自行生成 Ninja 描述 | 极佳 (`rules_go` 官方标准) |
| **Android 支持** | ⚠️ 仅限配合 Android AOSP 源码级构建 | 极佳 (大厂移动端巨仓首选方案) |
| **iOS 支持** | ❌ 完全不支持 | 极佳 (`rules_apple` 标准套件) |
| **Next.js 支持** | ❌ 完全不支持 | 支持 (`rules_js` 完美接管依赖图) |
| **最佳适用场景** | 编译 Chromium / Android AOSP 源码 | 跨语言、大团队的 Monorepo（单一巨仓） |

### 4.4 避坑提示：Bazel 的“终极代价”与轻量级替代方案

虽然 Bazel 理论上能够覆盖多种技术栈，但在决定引入前必须评估其较为陡峭的学习曲线。Bazel 的 Starlark 配置语法对习惯了原生包管理器的普通开发者并不友好，团队通常需要配备专职的构建工程师（Build Engineer）进行长期维护。

如果您的 Go、Android、iOS 和 Next.js 项目规模并 **没有达到几百万行代码、数百名开发者** 的级别，或者项目分散在不同的独立仓库中（Polyrepo），引入 Bazel 往往会带来过重的配置负担。此时，更推荐采用行业主流的**轻量级组合生态**：

* **Next.js 前端**：直接使用 Vercel 官方的 Turborepo 或 Nx（前端 Monorepo 的轻量化标准）。
* **Android 端**：使用官方默认的高级 Gradle 构建系统。
* **iOS 端**：使用 Xcode 原生构建并搭配 Swift Package Manager (SPM)。
* **Golang 后端**：直接使用原生原汁原味的 `go build`，或配合轻量的 Makefile / Taskfile 进行任务自动化调度。


如果是觉得 Bazel 是 Java 生态的工具太重（ JVM 消耗资源多，Bazel 学习成本高），可以考虑类似 Bazel 的构建工具: [Buck2 © 2026 Meta Platforms](https://buck2.build/) | [Please © 2024 Thought Machine](https://please.build/) | [Pantsbuild © Pants](https://www.pantsbuild.org/)。

---

## 五 总结

Siso 的出现标志着 Google 在追求超大型工程编译效能的道路上迈出了新的一步。它通过智能化地与远端分布式 API 交互，成功让 Chromium 这样庞大的工程项目在几分钟内完成编译。但对广大开发者而言，Siso 是一把为 Chromium 和 Android 源码量身定制的特定工具；而在面对包含 Go、移动端双端及 Next.js 等多元化、多技术栈的通用商业项目时，生态繁荣、跨语言接管能力极强的 Bazel 依然是分布式构建领域的行业主流解法。

---

## 六 参考

* **Siso 官方路线与自述文档**：访问 [Chromium 官方 Git 仓库的 Siso 主页](https://chromium.googlesource.com/build/+/HEAD/siso/README.md) 了解项目立项与基础定位。
* **Siso 替换 Ninja 与 Reclient 官方公告**：详见 Chromium-dev 开发者论坛上的技术通告 [PSA: Chromium's build system is switching from Ninja to Siso](https://groups.google.com/a/chromium.org/g/chromium-dev/c/v-WOvWUtOpg) 。
* **Chromium 文档的 Siso Tips**： https://chromium.googlesource.com/chromium/src/+/HEAD/docs/siso_tips.md。
* **Chromium 官方编译配置指南**：查阅 Chromium 源码文档库 [Linux Build Instructions](https://chromium.googlesource.com/chromium/src/+/main/docs/linux/build_instructions.md) 学习 `args.gn` 相关参数。
* **Bazel 各语言规则生态库**：前往官方开源软件仓了解 Go 语言规则集 [rules_go](https://github.com/bazel-contrib/rules_go) 以及 JavaScript/TypeScript 规则集 [rules_js](https://github.com/aspect-build/rules_js)。
