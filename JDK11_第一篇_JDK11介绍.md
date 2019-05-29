---
title: JDK11 | 第一篇 : JDK11 介绍
date: 2019-05-29 22:47:27:027
tags: [Java] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/5/29/16b02e75952425fa?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cee4eecf265da1bb679faed](https://juejin.im/post/5cee4eecf265da1bb679faed) 

## 一、简介

北京时间 2018年9 月 26 日，Oracle 官方宣布 Java 11 正式发布。这是 Java 大版本周期变化后的第一个长期支持版本，非常值得关注。从官网即可下载, 最新发布的 Java11 将带来 ZGC、Http Client 等重要特性。Java 11 新特性：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b02e75952425fa?imageView2/0/w/1280/h/960/ignore-error/1)

从时间节点来看，JDK 11 的发布正好处在 JDK 8 免费更新到期的前夕，同时 JDK 9、10 也陆续成为“历史版本”。JDK 11 将是一个 企业不可忽视的版本。

## 二、更新的细节

在过去的很多年中，Oracle 和 OpenJDK 社区提供了接近免费的午餐，导致人们忽略了其背后的海量工作和价值，这其中包括但不仅仅限于：最新的安全更新，如，安全协议等基础设施的升级和维护，安全漏洞的及时修补，这是 Java 成为企业核心设施的基础之一。大量的新特性、Bug 修复，例如，容器环境支持，GC 等基础领域的增强。很多生产开发中的 Hack，其实升级 JDK 就能解决了。不断改进的 JVM，提供接近零成本的性能优化…

### ZGC

JDK11 引入了两种新的 GC，其中包括也许是划时代意义的 ZGC，虽然其目前还是实验特性，但是从能力上来看，这是 JDK 的一个巨大突破，为特定生产环境的苛刻需求提供了一个可能的选择。例如，对部分企业核心存储等产品，如果能够保证不超过 10ms 的 GC 暂停，可靠性会上一个大的台阶，这是过去我们进行 GC 调优几乎做不到的，是能与不能的问题。

![](https://user-gold-cdn.xitu.io/2019/5/29/16b02e77fc015851?imageView2/0/w/1280/h/960/ignore-error/1)

对于 G1 GC，相比于 JDK 8，升级到 JDK 11 即可免费享受到：并行的 Full GC，快速的 CardTable 扫描，自适应的堆占用比例调整（IHOP），在并发标记阶段的类型卸载等等。这些都是针对 G1 的不断增强，其中串行 Full GC 等甚至是曾经被广泛诟病的短板，你会发现 GC 配置和调优在 JDK11 中越来越方便。

### Flight Recorder（JFR）

/*/*Flight Recorder（JFR）/*/*是 Oracle 刚刚开源的强大特性。**JFR** 是一套集成进入 JDK、JVM 内部的事件机制框架，通过良好架构和设计的框架，硬件层面的极致优化，生产环境的广泛验证，它可以做到极致的可靠和低开销。在 SPECjbb2015 等基准测试中，JFR 的性能开销最大不超过 1%，所以，工程师可以基本没有心理负担地在大规模分布式的生产系统使用，这意味着，我们既可以随时主动开启 JFR 进行特定诊断，也可以让系统长期运行 JFR，用以在复杂环境中进行“After-the-fact”分析。

在保证低开销的基础上，JFR 提供的能力可以应用在对锁竞争、阻塞、延迟，JVM GC、SafePoint 等领域，进行非常细粒度分析。甚至深入 JIT Compiler 内部，全面把握热点方法、内联、逆优化等等。JFR 提供了标准的 Java、C++ 等扩展 API，可以与各种层面的应用进行定制、集成，为复杂的企业应用栈或者复杂的分布式应用，提供 All-in-One 解决方案。而这一切都是内建在 JDK 和 JVM 内部的，并不需要额外的依赖，开箱即用。

### Low-Overhead Heap Profiling

它来源于 Google 等业界前沿厂商的一线实践，通过获取对象分配细节，为 JDK 补足了对象分配诊断方面的一些短板，工程师可以通过 JVMTI 使用这个能力增强自身的工具。

### HTTP/2 Client API

新的 HTTP API 提供了对 HTTP/2 等业界前沿标准的支持，精简而又友好的 API 接口，与主流开源 API（如，Apache HttpClient， Jetty， OkHttp 等）对等甚至更高的性能。与此同时它是 JDK 在 Reactive-Stream 方面的第一个生产实践，广泛使用了 Java Flow API 等，终于让 Java 标准 HTTP 类库在扩展能力等方面，满足了现代互联网的需求。

### Transport Layer Security (TLS) 1.3

就是安全类库、标准等方面的大范围升级，它还是中国安全专家范学雷所领导的 JDK 项目，完全不同于以往的修修补补，是个非常大规模的工程。

### Dynamic Class-File Constants

动态 class 文件常量。扩展了 Java class 文件格式，支持一种新的常量池形式：CONSTANT_Dynamic。

### Improve Aarch64 Intrinsics

主要是针对 ARM Aarch64 架构的优化，比如提供优化的 sin、cos 等函数。

### Epsilon: A No-Op Garbage Collector(Experimental)

无操作的垃圾收集器。Epsilon 是一个特殊的垃圾收集器，只处理内存分配，不负责回收。一旦堆耗尽，就关闭 JVM。

听上去这个收集器好像没什么意义。不过它还是有不少用处的。比如：

性能测试。GC 会影响性能，有了这么一个几乎什么都不干的 GC，我们可以过滤掉 GC 带来的影响因素。还有些性能因素不是 GC 引入的，比如编译器变换，利用 Epsilon GC，我们可以对比。就像生物学里做实验，我们可以用它做一个对照组。

另外还有内存压力测试、VM接口测试等。

### Launch Single-File Source-Code Programs

### Unicode 10

升级现有 API 支持 Unicode 10。Java SE 10 实现的是 Unicode 8.0。与 Java 10 相比，Java 11 多支持 16 018 个新字符，10 种新的文字类型。

### Nest-Based Access Control

基于嵌套的访问控制。Java 11 引入了 nest 的概念，这是一个新的访问控制上下文（context），逻辑上处于同一代码实体中的类，尽管会被编译为不同的 class 文件，但是可以访问彼此的 private 成员，不再需要编译器插入辅助访问的桥方法。

### Dynamic Class-File Constants

动态 class 文件常量。扩展了 Java class 文件格式，支持一种新的常量池形式：CONSTANT_Dynamic。

### Remove the Java EE and CORBA Modules

将 Java SE 9 中标记为废弃的 Java EE 和 CORBA 正式从 Java SE 平台中删除。

### Launch Single-File Source-Code Programs

支持运行单个文件中的源代码。在刚学习 Java 或者编写小的工具程序时，我们一般要先用 javac 编译源文件，再用 java 命令运行。有了这个功能，我们可以直接用 java 命令运行源程序。就像这样：
```
java HelloWorld.java
```

### Deprecate the Nashorn JavaScript Engine

废弃 Nashorn JavaScript 脚本引擎、API 和 jjs 工具。Nashorn 是在 JDK 8 中引入的，当时完整实现了 ECMAScript-262 5.1。不过随着 ECMAScript 的演进加快，Nashorn 维护越来越困难。

### Deprecate the Pack200 Tools and API

废弃了 pack200 和 unpack200 工具，以及 java.util.jar 包中的 Pack200 API。

欢迎扫码或微信搜索公众号《程序员果果》关注我，关注有惊喜~
![](https://user-gold-cdn.xitu.io/2019/4/11/16a09fab998c3fb5?imageView2/0/w/1280/h/960/ignore-error/1)

