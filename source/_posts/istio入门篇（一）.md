---
title: istio入门篇（一）
date: 2025-04-18 11:39:02
tags: istio
categories: istio
---
#  一、背景

一直以来“微服务”都是一个热门的词汇，在各种技术文章、大会上，关于微服务的讨论和主题都很多。对于基于 Dubbo、SpringCloud 技术体系的微服务架构，已经相当成熟并被大家所知晓，但伴随着互联网场景的复杂度提升、业务快速变更以及快速响应，如何快速、稳定、高效的应对变幻莫测的业务市场需求，这类技术体系（如：Spring Cloud）的传统微服务架构就变得力不从心，此时微服务架构再次升级，将服务网格作为了新一代微服务架构。

微服务，也称之为微服务架构，是一种架构风格，相比单体应用，它将应用程序拆分为一组服务，并将这些服务组合起来来完成整个复杂的业务功能。下面这些特征就能高度反映出它的价值所在：

- 高度可维护和可测试性
- 松耦合
- 独立部署
- 围绕业务能力进行组织
- 小团队拥有

简单的回顾完微服务架构的概念，我们一起看看新一代微服务架构是如何诞生的。

### **基于 Spring Cloud 的微服务体系**

下面这张图是基于 Spring Cloud 技术体系的微服务架构图：



![img](https://gitee.com/ljh00928/csdn/raw/master/img/e9f83fc160d0108ca77cf6b909ec84e1.png)

- 实现：所有微服务都需要将自身注册到注册中心（如：Consul、Eureka 等），来完成服务间的相互调用。每个微服务都必须依赖 Spring Cloud 组件（即：在 pom.xml 中引入），业务逻辑和 Spring Cloud 组件共生在同一个服务中。

还记得 Spring Cloud 相关组件版本**升级**时的烦恼么？为了使用新版本中的某个特性，或者解决旧版本中存在的漏洞,Spring Cloud 版本升级屡见不鲜，一不留神就会出现版本依赖冲突、启动不了等等问题，升级完还得安排测试人员测试验证。技术含量不高，但确实招人烦啊。

再完美的程序，也避免不了零 bug。上线之后，随着系统使用场景的多样性，将逐步会暴露出一些问题，而出现问题就得解决问题，并**小心翼翼**安排上线，这一系列过程，想必各位肯定深有感触，各有故事。用“小心翼翼”来形容这一过程决不夸张，因为一个小小的改动可能会影响到其它，甚至整个系统，这锅谁都不太想背，**能不改打死都不改的原则一直是不愿被打破的壁垒**。

在传统行业（如：银行），由于系统的多样性、庞大、复杂性，全部加入微服务行列是不现实的，**新老系统共存**是一种最为常见的现象。而共存系统间的治理、运维等成了老大难问题

### **传统微服务架构面临的挑战**

面对上述暴露出的问题，并在传统微服务架构下，经过实践的不断冲击，面临了更多新的挑战，综上所述，产生这些问题的原因有以下这几点：

- 过于绑定特定技术栈 当面对异构系统时，需要花费大量精力来进行代码的改造，不同异构系统可能面临不同的改造。
- 代码侵入度过高 开发者往往需要花费大量的精力来考虑如何与框架或 SDK 结合，并在业务中更好的深度融合，对于大部分开发者而言都是一个高曲线的学习过程。
- 多语言支持受限 微服务提倡不同组件可以使用最适合它的语言开发，但是传统微服务框架，如 Spring Cloud 则是 Java 的天下，多语言的支持难度很大。这也就导致在面对异构系统对接时的无奈，或选择退而求其次的方案了。
- 老旧系统维护难 面对老旧系统，很难做到统一维护、治理、监控等，在过度时期往往需要多个团队分而管之，维护难度加大。

上述这些问题都是在所难免，我们都知道技术演进来源于实践中不断的摸索，将功能抽象、解耦、封装、服务化。 随着传统微服务架构暴露出的这些问题，将迎来新的挑战，让大家纷纷寻找其他解决方案。

**Spring Cloud 微服务架构和 Service Mesh 微服务架构**



![img](https://gitee.com/ljh00928/csdn/raw/master/img/fd8be54cbe7b3b978bdcbfccc40b11f9.png)

**为了解决微服务框架的侵入性问题，我们引入服务网格。**

官方文档： [Istioldie 1.17 / 架构](https://istio.io/v1.17/zh/docs/ops/deployment/architecture/)

参考地址：[全方位解读服务网格（Service Mesh）的背景和概念-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1876488)

### 迎来新一代微服务架构

为了解决传统微服务面临的问题，以应对全新的挑战，微服务架构也进一步演化，最终催生了服务网格（Service Mesh）的出现，迎来了新一代微服务架构，也被称为下一代微服务。为了更好地理解 Service Mesh 的概念和存在的意义，让我们我们来回顾一下这一演进过程中的四个阶段。



![img](https://gitee.com/ljh00928/csdn/raw/master/img/2ec1597668423d592221ec635ff82f20.png)

- 耦合阶段：高度耦合、重复实现、维护困难，在耦合架构设计中体现的最为突出，单体架构就是典型的代表。
- 公共 SDK：让基础设施功能设计成为公共 SDK，提高利用率，是解藕最有效的途径，比如 Spring Cloud 就是类似的方式。但学习成本高、特定语言实现，却将一部分人拦在了门外。
- Sidecar 模式：再次深度解藕，不单单功能解藕，更从跨语言、更新发布和运维等方面入手，实现对业务服务的零侵入，更解藕于开发语言和单一技术栈，实现了完全隔离，为部署、升级带来了便利，做到了真正的基础设施层与业务逻辑层的彻底解耦。另一方面，Sidecar 可以更加快速地为应用服务提供更灵活的扩展，而不需要应用服务的大量改造。
- Service Mesh：把 Sidecar 模式充分应用到一个庞大的微服务架构系统中来，为每个应用服务配套部署一个 Sidecar 代理，完成服务间复杂的通信，最终就会得到一个的网络拓扑结构，这就是 Service Mesh，又称之为“服务网格“。它从本质上解决了传统微服务所面临的问题。

# 二、服务网格介绍

### 什么是服务网格

在过去的几十年中，我们已经看到了单体应用程序开始拆分为较小的应用程序。此外，诸如Docker之类的容器化技术和诸如Kubernetes之类的编排系统加速了这一变化。

尽管在像Kubernetes这样的分布式系统上采用微服务架构有许多优势，但它也具有相当的复杂性。由于分布式服务必须相互通信，因此我们必须考虑发现，路由，重试和故障转移。

还有其他一些问题，例如安全性和可观察性，我们还必须注意以下问题：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/371bc18dbb634a97b40663899fb063d2.png)

现在，在每个服务中建立这些通信功能可能非常繁琐，尤其是当服务范围扩大且通信变得复杂时，更是如此。这正是服务网格可以为我们提供帮助的地方。基本上，服务网格消除了在分布式软件系统中管理所有服务到服务通信的责任。

服务网格能够通过一组网络代理来做到这一点。本质上，服务之间的请求是通过与服务一起运行但位于基础结构层之外的代理路由的：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/f61dc77d33834fae95aa37afe2135804.png)


这些代理基本上为服务创建了一个网状网络——因此得名为服务网格！通过这些代理，服务网格能够控制服务到服务通信的各个方面。这样，我们可以使用它来解决分布式计算的八个谬误，这是一组断言，描述了我们经常对分布式应用程序做出的错误假设。

### 服务网格的特征

现在，让我们了解服务网格可以为我们提供的一些功能。请注意，实际功能列表取决于服务网格的实现。但是，总的来说，我们应该在所有实现中都期望其中大多数功能。

我们可以将这些功能大致分为三类：流量管理，安全性和可观察性。

**流量管理**

服务网格的基本特征之一是流量管理。这包括动态服务发现和路由。尤其影子流量和流量拆分功能，这些对于实现金丝雀发布和A/B测试非常有用。

由于所有服务之间的通信都是由服务网格处理的，因此它还启用了一些可靠性功能。例如，服务网格可以提供重试，超时，速率限制和断路器。这些现成的故障恢复功能使通信更加可靠。

**安全性**

服务网格通常还处理服务到服务通信的安全性方面。这包括通过双向TLS（mTLS）强制进行流量加密，通过证书验证提供身份验证以及通过访问策略确保授权。

服务网格中还可能存在一些有趣的安全用例。例如，我们可以实现网络分段，从而允许某些服务进行通信而禁止其他服务。而且，服务网格可以为审核需求提供精确的历史信息。

**可观察性**

强大的可观察性是处理分布式系统复杂性的基本要求。由于服务网格可以处理所有通信，因此正确放置了它可以提供可观察性的功能。例如，它可以提供有关分布式追踪的信息。

服务网格可以生成许多指标，例如延迟，流量，错误和饱和度。此外，服务网格还可以生成访问日志，为每个请求提供完整记录。这些对于理解单个服务以及整个系统的行为非常有用。

# 三、istio的介绍

Istio是最初由IBM，Google和Lyft开发的服务网格的开源实现。它可以透明地分层到分布式应用程序上，并提供服务网格的所有优点，例如流量管理，安全性和可观察性。

它旨在与各种部署配合使用，例如本地部署，云托管，Kubernetes容器以及虚拟机上运行的服务程序。尽管Istio与平台无关，但它经常与Kubernetes平台上部署的微服务一起使用。

从根本上讲，Istio的工作原理是以Sidcar的形式将Envoy的扩展版本作为代理布署到每个微服务中：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/a2669e2386564760962281efd71f9ba3.png)

该代理网络构成了Istio架构的数据平面。这些代理的配置和管理是从控制平面完成的：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/ef015ccc0b1c4a848d48aeb274d13425.png)

控制平面基本上是服务网格的大脑。它为数据平面中的Envoy代理提供发现，配置和证书管理。

当然，只有在拥有大量相互通信的微服务时，我们才能体现Istio的优势。在这里，sidecar代理在专用的基础架构层中形成一个复杂的服务网格：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/92b68553257d4277bfa2d044ae874055.png)



Istio在与外部库和平台集成方面非常灵活。例如，我们可以将Istio与外部日志记录平台，遥测或策略系统集成。

# 四、Istio组件

我们已经看到，Istio体系结构由数据平面和控制平面组成。此外，还有几个使Istio起作用的核心组件。

在本节中，我们将详细介绍这些核心组件。

**Envoy**：一个高性能的代理，负责处理服务之间的通信。
 **Pilot**：负责配置Envoy代理，提供服务发现、负载均衡、路由规则等功能。
 **Citadel**：负责证书管理和身份认证。
 **Galley**：负责配置管理，验证和分发Istio的配置。

**数据平面**：由Envoy代理组成，负责处理服务之间的通信。
 **控制平面**：由Pilot、Citadel和Galley组成，负责配置和管理数据平面。

### **数据平面**

Envoy代理以Sidecar的形式部署在每个服务实例旁边，拦截所有的入站和出站流量。Envoy代理负责执行流量管理、安全性和可观测性等功能。

Istio的数据平面主要包括Envoy代理的扩展版本。Envoy是一个开源边缘和服务代理，可帮助将网络问题与底层应用程序分离开来。应用程序仅向localhost发送消息或从localhost接收消息，而无需了解网络拓扑。

Envoy的核心是在OSI模型的L3和L4层运行的网络代理。它通过使用可插入网络过滤器链来执行连接处理。此外，Envoy支持用于基于HTTP的流量的附加L7层过滤器。而且，Envoy对HTTP/2和gRPC传输具有一流的支持。

Istio作为服务网格提供的许多功能实际上是由Envoy代理的基础内置功能启用的：

- 流量控制：Envoy通过HTTP，gRPC，WebSocket和TCP流量的丰富路由规则启用细粒度的流量控制应用
- 网络弹性：Envoy包括对自动重试，断路和故障注入的开箱即用支持
- 安全性：Envoy还可以实施安全策略，并对基础服务之间的通信应用访问控制和速率限制

Envoy在Istio上表现出色的另一个原因之一是它的可扩展性。Envoy提供了基于WebAssembly的可插拔扩展模型。这在定制策略执行和遥测生成中非常有用。此外，我们还可以使用基于Proxy-Wasm沙箱API的Istio扩展在Istio中扩展Envoy代理。

### **控制面**

控制平面负责配置和管理数据平面。Pilot负责将路由规则、服务发现等信息分发给Envoy代理；Citadel负责证书管理和身份认证；Galley负责验证和分发Istio的配置。

如上所述，控制平面负责管理和配置数据平面中的Envoy代理。在Istio架构中，控制面核心组件是istiod，Istiod负责将高级路由规则和流量控制行为转换为特定于Envoy的配置，并在运行时将其传播到Sidercar。

如果我们回顾一下Istio控制平面的架构，将会注意到它曾经是一组相互协作的独立组件。它包括诸如用于服务发现的Pilot，用于配置的Galley，用于证书生成的Citadel以及用于可扩展性的Mixer之类的组件。由于复杂性，这些单独的组件被合并为一个称为istiod的单个组件。

从根本上来说，istiod仍使用与先前各个组件相同的代码和API。例如，Pilot负责抽象特定于平台的服务发现机制，并将其合成为Sidecar可以使用的标准格式。因此，Istio可以支持针对多个环境（例如Kubernetes或虚拟机）的发现。

此外，istiod还提供安全性，通过内置的身份和凭据管理实现强大的服务到服务和最终用户身份验证。此外，借助istiod，我们可以基于服务身份来实施安全策略。该过程也充当证书颁发机构（CA）并生成证书，以促进数据平面中的相互TLS（MTLS）通信。

# 五、Istio工作原理

我们已经了解了服务网格的典型特征是什么。此外，我们介绍了Istio架构及其核心组件的基础。现在，是时候了解Istio如何通过其架构中的核心组件提供这些功能了。

我们将专注于我们之前经历过的相同类别的功能。

### **流量管理**

我们可以使用Istio流量管理API对服务网格中的流量进行精细控制。我们可以使用这些API将自己的流量配置添加到Istio。此外，我们可以使用Kubernetes自定义资源定义（CRD）定义API资源。帮助我们控制流量路由的关键API资源是虚拟服务和目标规则：

![img](https://gitee.com/ljh00928/csdn/raw/master/img/371856c6dfa141aa89adaf610801b2ff.png)

基本上，虚拟服务使我们可以配置如何将请求路由到Istio服务网格中的服务。因此，虚拟服务由一个或多个按顺序评估的路由规则组成。评估虚拟服务的路由规则后，将应用目标规则。目标规则有助于我们控制到达目标的流量，例如，按版本对服务实例进行分组。

### 安全性

Istio为每个服务提供身份。与每个Envoy代理一起运行的Istio代理与istiod一起使用以自动进行密钥和证书轮换

![img](https://gitee.com/ljh00928/csdn/raw/master/img/12d9f2f12a3f4da69cf295282012cd23.png)



Istio提供两种身份验证——对等身份验证和请求身份验证。对等身份验证用于服务到服务的身份验证，其中Istio提供双向TLS作为全栈解决方案。请求身份验证用于最终用户身份验证，其中Istio使用自定义身份验证提供程序或OpenID Connect（OIDC）提供程序提供JSON Web令牌（JWT）验证。

Istio还允许我们通过简单地将授权策略应用于服务来实施对服务的访问控制。授权策略对Envoy代理中的入站流量实施访问控制。这样，我们就可以在各种级别上应用访问控制：网格，命名空间和服务范围。

### 可观察性

Istio为网格网络内的所有服务通信生成详细的遥测，例如度量，分布式跟踪和访问日志。Istio生成一组丰富的代理级指标，面向服务的指标和控制平面指标。

之前，Istio遥测体系结构将Mixer作为核心组件。但是从Telemetry v2开始，混音器提供的功能已替换为Envoy代理插件

![img](https://gitee.com/ljh00928/csdn/raw/master/img/1287d0c535144f6fb36db31ed55fa673.png)

此外，Istio通过Envoy代理生成分布式跟踪。Istio支持许多跟踪后端，例如Zipkin，Jaeger，Lightstep和Datadog。我们还可以控制跟踪速率的采样率。此外，Istio还以一组可配置的格式生成服务流量的访问日志。

# 六、安装istio

k8s和istio版本对应表

[Istio / Supported Releases](https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases)

查看k8s版本

```
[root@master231 ~]# kubectl version --short
Client Version: v1.23.17
Server Version: v1.23.17
```

下载istio

```
下载最新版本：curl -L https://istio.io/downloadIstio | sh -
下载指定版本：curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.17.3 TARGET_ARCH=x86_64 sh -
```

配置环境变量

```
[root@master231 ~]# cd istio-1.17.3/bin

[root@master231 ~]# vim .bashrc 
...
export PATH=/root/istio-1.17.3/bin:$PATH

[root@master231 ~]# source .bashrc
```

查看安装模式

```
[root@master231 ~]# istioctl profile list
Istio configuration profiles:
    ambient
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote
```

在安装 Istio 时所能够使用的内置配置文件。这些配置文件提供了对 Istio 控制平面和 Istio 数据平面 Sidecar 的定制内容。

- default: 根据 IstioOperator API 的默认设置启动组件。 建议用于生产部署和 Multicluster Mesh 中的 Primary Cluster。 您可以运行 istioctl profile dump 命令来查看默认设置。
- demo： 这一配置具有适度的资源需求，旨在展示 Istio 的功能。 它适合运行 Bookinfo 应用程序和相关任务。 这是通过快速开始指导安装的配置。 此配置文件启用了高级别的追踪和访问日志，因此不适合进行性能测试。
- minimal： 与默认配置文件相同，但只安装了控制平面组件。 它允许您使用 Separate Profile 配置控制平面和数据平面组件(例如 Gateway)。
- remote： 配置 Multicluster Mesh 的 Remote Cluster。
- empty： 不部署任何东西。可以作为自定义配置的基本配置文件。
- preview： 预览文件包含的功能都是实验性。这是为了探索 Istio 的新功能。不确保稳定性、安全性和性能（使用风险需自负）。

**标注 ✔ 的组件安装在每个配置文件中：**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/655d95fc8f19495e930095d69ef6f733.png)

安装istio

```
[root@master231 ~]# istioctl install --set profile=demo -y
```

查看配置

```
[root@master231 ~]# istioctl profile dump demo|default|minimal|...
```

添加自动补全

```
[root@master231 tools]# source /root/istio-1.17.3/tools/istioctl.bash
```

创建命名空间，让所有pod注入边车形式

```
[root@master231 ~]# kubectl create ns fox
namespace/fox created
```

指示 Istio 在部署应用的时候，在指]定名称空间下。为fox命名空间打上标签 `istio-injection=enabled`。自动注入 Envoy 边车代理

```
[root@master231 ~]# kubectl label namespace fox istio-injection=enabled
namespace/fox labeled
```

# 七、案例部署

如果使用上面步骤安装的istio，那么就已经安装了Bookinfo这个应用。该应用由四个单独的微服务构成。

这个应用模仿在线书店的一个分类，显示一本书的信息。 页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- `productpage`. 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`. 这个微服务中包含了书籍的信息。
- `reviews`. 这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`. 这个微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息

![img](https://gitee.com/ljh00928/csdn/raw/master/img/14acff20f6724e8cad8da5917381dc04.png)

要在 Istio 中运行这一应用，无需对应用自身做出任何改变。 您只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。 最终的部署结果将如下图所示

![img](https://gitee.com/ljh00928/csdn/raw/master/img/b0b52de1c84d4fcda25d4d674289a4a8.png)

所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。

### 启动应用服务

使用手动注入Sidecar

```
[root@master231 ~]# kubectl apply -f <(istioctl kube-inject -f /root/istio-1.17.3/samples/bookinfo/platform/kube/bookinfo.yaml)
```

查看pod状态

```
[root@master231 ~]# kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-754db4b55f-vcbhq       2/2     Running   0          19m
productpage-v1-64c6fcc6f6-b6552   2/2     Running   0          19m
ratings-v1-5d4d5694ff-g7x5s       2/2     Running   0          19m
reviews-v1-6878588b96-p6jht       2/2     Running   0          19m
reviews-v2-6dfc59845c-hx4t8       2/2     Running   0          19m
reviews-v3-5b5f87dd46-bq566       2/2     Running   0          19m
```

查看svc状态 

```
[root@master231 ~]# kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.200.90.164   <none>        9080/TCP   19m
kubernetes    ClusterIP   10.200.0.1      <none>        443/TCP    28d
productpage   ClusterIP   10.200.84.58    <none>        9080/TCP   19m
ratings       ClusterIP   10.200.90.62    <none>        9080/TCP   19m
reviews       ClusterIP   10.200.201.22   <none>        9080/TCP   19m
```

确认 Bookinfo 应用是否正在运行，请在某个 Pod 中用 curl 命令对应用发送请求，例如 ratings：

```
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

出现这个说明bookinfo应用没问题

![img](https://gitee.com/ljh00928/csdn/raw/master/img/b187b91fd8e7492a9e8cb0038f93dfe1.png)

### 创建Gateway

大家可以先看下案例中Gateway和VirtualService是怎么写的

```
[root@master231 istio-1.17.3]# cat samples/bookinfo/networking/bookinfo-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

Istio Ingress Gateway，并与应用程序关联

```
[root@master231 istio-1.17.3]# kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml 
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

查看网关

```
[root@master231 istio-1.17.3]# kubectl get gw
NAME               AGE
bookinfo-gateway   26s

```

根据文档设置访问网关的 INGRESS_HOST 和 INGRESS_PORT 变量。确认并设置。 在Istio 的安装文档中，我已经通过NodePort 方式来暴露istio-ingressgateway 服务，现在根据如下命令来获取

```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

获取 ingress IP 地址

```
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')
```

设置并获取GATEWAY_URL：

```
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

[root@master231 istio-1.17.3]# echo $GATEWAY_URL
10.0.0.232:31977
```

访问

```
http://10.0.0.232:31977/productpage
```

每次请求都会打到不同的reviews。大家可以刷新测试

![img](https://gitee.com/ljh00928/csdn/raw/master/img/b7bd621ba52346e592b646eb9b9df3d7.png)