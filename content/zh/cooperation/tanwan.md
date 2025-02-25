---
type: docs
title: "CloudWeGo 在贪玩游戏 SDK 接口上的应用实践"
linkTitle: "贪玩游戏"
weight: 4
---

> 
> 3月，CloudWeGo Day  邀请了贪玩游戏、数美科技、字节跳动业务部门的一线架构师，为大家分享，在Java 转 go 场景下、企业高可用性挑战等场景下，如何通过 CloudWeGo 来落地微服务架构。本文为 **贪玩游戏技术经理 李华松** 分享内容。
> 
> **🔗 回放链接：** https://juejin.cn/live/cloudwegoday002
> 

## 嘉宾介绍

![tanwan1](/img/users/tanwan/tanwan1.png)

本次分享主要分为以下四部分内容：

1.  **遇到的架构问题**。QPS上不去，业务瞬时高并发，扩容慢。
1.  **语言架构的探索选型**。原生go，go框架。
1.  **CloudWeGo 架构的落地**。牛刀小试，积累经验，迅速扩展，修复问题。
1.  **给相似业务的参考建议**。按团队思想、水平安排推进节奏，做好微服务拆分， 运维快学，善用外部资源，善用工具做好监控。


## 遇到的架构问题

### 公司业务

在这之前，我先简单介绍一下我们的公司“贪玩游戏”。我们公司贪玩游戏在业内比较知名，主要是因为玩家对我们的厚爱；其次，我们还有许多出圈的文案，例如“一刀999”、“是兄弟就来砍我”等。

1.  **公司业务重心：** 我们致力于让玩家顺畅地进入游戏，享受游戏的乐趣。为玩家提供注册、登录、充值以及游戏内的各种服务。
1.  **公司业务性质：** 我们是一家买量公司，业务上频繁开服并推送大量流量，因此对框架性能的要求也越来越高。
1.  **公司业务需求**：我们技术团队专注于快速开发和迭代，不断跟进市场和运营的需求，不断探索创新，以满足客户的需求。

### 遇到的架构问题

1.  **此前使用的 php-fpm 的** **QPS** **瓶颈值低。** 单台阿里云ecs机器(8c16G或16c32G) php-fpm 在业务侧压测及实际表现 QPS 在400-600之间。

1.  **瞬时高并发场景下，php** **-** **fpm 处理能力明显不够用**。

    - 场景1：**游戏大推** **。** 比如邀请了一个很受欢迎的代言人，在今日头条、抖音、广点通、微信朋友圈等各个媒体里大推广告时，如果我们的平台无法承受压力，可能会导致资金浪费。
    - 场景2：**合作方集中推送游戏数据** **。** 因为我们是平台类应用，会关注用户各个方面的数据，合作方也会推送过来。合作方推送的数据通常是默认为 no problem，即合作方推送多少你都可以接多少。在这种情况下，如果我们的 php-fpm **处理能力**较低，合作方的观感可能会受到影响。
    - 场景3：**合作方游戏更新大批量玩家验证 token、重登录**。游戏通常每两周更新一次，更新时如果程序重新启动，可能会导致游戏掉线并重新启动。如果有 token，则需要提交 token 进行验证，同时需要具备高并发能力以应对大量瞬时上线的请求。
    -  场景4：**接口被刷**。因为游戏中有一些生态，有些玩家会在游戏生态中找到存在或生存的方法，因此会储备大量账号。如果接口被刷，我们会有风控措施，但万一这些刷量的大量请求到达服务器时，我们也应该能够承受。

1.  **运维扩展时间成本高，不灵活。**
例如处理不当或不足，可能导致单台服务器出现问题。为了解决这个问题，我们需要不断地增加服务器数量，这也会带来时间成本。如果完成了以下三个步骤，通常需要7分钟，这可能会导致玩家流失。
    -  3分钟，创建ecs服务
    -  3分钟，部署环境+同步代码+基本测试
    -  1分钟，上线现网环境

### 语言架构的探索选型


**原生 Go 实践与应用**

**Golang**

Golang 是一种注重高效和并发编程的编程语言，提供了内存安全、代码简洁、跨平台和易于部署等优点，适用于大规模、高并发的网络服务和系统编程。

我曾在15、16 年使用 Go 语言，当时大多数人都写的是原生的 Go 代码。因此，我们在这个场景下尝试将 Go 语言用于媒体点击，即投放广告时，广告媒体会向我们发送各种日志，我们从中收集信息并进行匹配分析。

**背景**

媒体点击下发服务高峰时能达到6、7kQPS，单机 php-fpm 性能难以支撑这么高的并发量，只能通过不断增加服务器资源成本来解决。而 Go 在并发这块有出色的性能表现，能极大地节省资源成本。

在接收到日志信号的过程中，如果我们使用 php-fpm 来进行记录，它可能会达到几百个 QPS。因此，需要部署数十台服务器才能支撑如此高的 QPS，我们认为这不应该耗费这么多资源。因此，我们将 Go 语言用于实现一个简单的替换，即使用原生的 Go 语言编写一个 HTTP 服务，将日志请求转发到 Go 中，并将这些日志写到指定的 channel 中，然后返回。channel 后边有一个定时器不断地将信道里的日志写到指定的地方。 

**使用**

通过使用 net/http 标准库 快递搭建一个 http 服务器，下发的数据通过 Channel 操作实时写入日志文件，然后通过 filebeat 采集同步到 kafka 进行数据清洗，重构后单台8核16g内存的服务器可以扛住 **1w+ QPS**。

![tanwan2](/img/users/tanwan/tanwan2.png)
 

**业务重构**

接下来，将介绍整个实现流程。

1.  **nginx** **路由转发**。首先就是 nginx 这一层，我们设置一个 location 转到新程序监听的 unix socket 上面。因为我们之前在ecs上面运行，部署监听是在 unix socket 上面，这个会比监听在端口更高效。

![tanwan3](/img/users/tanwan/tanwan3.png)

2.  **channel操作**。这个数据我们就塞到这个 channel 里面去了。

![tanwan4](/img/users/tanwan/tanwan4.png)

3.  **定时器执行** **IO** **操作。** 这个读数据出来，然后把它写入文件下边。

![tanwan5](/img/users/tanwan/tanwan5.png)

4.  **压测** **效果 1w+** **QPS** **。** 之前我们只有几百个 QPS，现在基本上能够达到1万个 QPS，即一台服务器能顶以前好几十台。对于小业务场景而言，这个表现非常好。

![tanwan6](/img/users/tanwan/tanwan6.png)

**token 验证接口**

还有一个使用场景，即我们使用的 token 验证接口。每逢到周四的某游戏更新日就会将玩家全部踢下线，等更新完成之后会有大量玩家在短时间内大量请求贪玩平台，我们的二次验证接口会校验这个 token 是否合法正常或是否已失效，认证不通过则需要重新登录。在短时间请求会批量涌过来，会导致原合法 token 变成异常返回，导致玩家会重复登陆，在下面两个图中表现出我们的 QPS 一直都居高不下。

假设因为程序响应不及时和返回异常时，会导致玩家不断地尝试，导致 QPS 呈倍数增长，多次请求后可能会触发雪崩。而前置的 waf 也会将过多请求进行拦截，玩家体验相当不好。

![tanwan7](/img/users/tanwan/tanwan7.png)

在 waf 拦截之后，如果后续操作没有完成，它仍会保持这种请求量高的状态，高位不断抖动。

**04-28 事故**

4月28日事故中，更新的游戏版本出现一些问题导致有两次更新， 而每次更新后 QPS 数又重新起来了，而这次的用户量更大、请求量更大，玩家更加着急了。

![tanwan8](/img/users/tanwan/tanwan8.png)

**处理方案**

从这个事件看得出，服务端的表现并不ok。我们要解决该问题，我们看一下我们的处理方式。

首先，php-fpm 确实存在没办法 hold 住高并发的问题，我们可能就做一些限流，比如说像地铁一样，你的车就只能载这么多人，那我就限流一下，不能不断地往里边塞。

1.  **nginx 加入限流**，同一时刻只服务有限次数，另外的被放弃，用户后一次尝试可能成功，只要玩家一直试，洪峰总会过去。在 nginx 上设置限流，但限流还是有部分用户在外边进不来，用户可能一直等。但我们不能让忠实玩家一直都在等，那我们总要解决这个问题。

![tanwan9](/img/users/tanwan/tanwan9.png)

2.  **使用 Go 重写相关逻辑平替掉该请求**。根据我们当时的经验，我们决定使用 Go 语言重写该业务逻辑。我们将其作为一个小项目进行重写，重写过程与之前的操作大致相同，将业务逻辑写进代码中即可。我们很快完成了重写，可以看到后续的表现。我们遇到一瞬间有那么多的请求过来，这是正常的表现，我们及时接住了它们，接住这些请求，QPS 就会下去了。

![tanwan10](/img/users/tanwan/tanwan10.png)

**05-13平替后的效果**

看下图的 Nginx QPS 可以看到新的 Token 认证接口可以承受量有很大的突破，保证了接口的可用性，而第二个图是登录接口 QPS，也反向证明了提高 Token 认证可用性，不会导致其它连锁反应，到此这个问题就解决了。

![tanwan11](/img/users/tanwan/tanwan11.png)

扛住了，请求量高起只一会，没有引发雪崩

![tanwan12](/img/users/tanwan/tanwan12.png)

同时段 token 验证到失效了，要再次登录的请求量

  
### 语言架构的探索选型

### 我们的选择 CloudWeGo

那到目前这里，我们看到了我们自己的一些探索，在与语言的探索中，我们探索了 Go，明显看到Go 处理的在某些场景更加牛、更加高效。因为我们知道这个很牛，我们要在更多的场景使用它，我们团队需要统一的行动，大家一起把这个事情做得更好，那么就需要选一个框架。

我们进行了一些选型，业内有很多选择，在当前的情况下，知道业内强劲的企业，在它早期的时候会有一些分享，奠定这个行业面对相似的技术难题时的基本选择，会形成一个影响，一个参考。

业内强劲企业在其早期阶段的分享的影响:

1.  腾讯微信技术总监周颢：一亿用户增长背后的架构秘密
1.  今日头条 Go 建千亿级微服务的实践

我们会根据我们的经验来评估他们之前是否做过类似场景的开发，并查看他们是否开源了相关内容。在这种情况下，我们发现了字节跳动开源了 CloudWeGo 框架。

###  **高性能的系列开源组件**

如果仅从普通框架的角度来看待它，一般认为上层建筑更为重要，但 CloudWeGo 会进行底层的优化，专注于打造高性能的框架和中间件。

**例如 Netpoll 和 Sonic 等，在精益求精、降本增效环境下极为重要**。因此我们决定使用当前我们认为最好且底蕴最深厚的开源项目。

![tanwan13](/img/users/tanwan/tanwan13.png)

###  快速开发，扩展性强

我们的诉求是，即可实现简易版单体应用开发，又可实现微服务架构拆分和高性能通信。

我们用的过程中也发现，CloudWeGo 能够快速开发，扩展性很强，简单几行代码就可以把相关功能做出来了。

以 HTTP 框架为例，从下图也可以看出，Hertz 的性能表现确实是相当不错的。

![tanwan14](/img/users/tanwan/tanwan14.png)

###  专业辅助，社区火热

此外，CloudWeGo 社区也很活跃。有专业技术人士指引辅导，且基本2周一次社区开发者例会。

我们整个团队要统一思想去行动，我们要在更多的业务场景上去用 go，于是就选择了 CloudWeGo 框架，主要是基于它的高性能、快速开发，以及有专业的辅助。

没有最好的框架，只有最适合的框架！恰逢开源，热烈拥抱。

## CloudWeGo 架构的落地

### 首次核心业务实践

**选中 SDK-LOGIN 手游登录接口**

我们团队希望寻找一个场景来应用 CloudWeGo 架构，同时借此熟练并掌握这一套技术，以便更好地推广和应用。因此，我们选择了一个场景，即我们首次核心业务的实践，用于开发我们的手游 SDK 登录接口。

手游登录接口是整个游戏 SDK 业务流程中核心的功能，且具备较重的业务逻辑，同时我们认为这接口有较大的优化空间和实践意义。

**平行业务拆分**

化繁为简，按业务水平拆分成多个 Service，简化每个 Service 的业务复杂度同时增加并行处理的可能性。

我们可以看到下图中，上边部分是我们之前在 PHP 的一个实践，即在用户端调用 Api 接口时，我们接口服务器也调用其它服务Api接口。一定程度上等同于微服务，故而在新接口上我们也是按照这个理念去拆分服务，然后中间做了个注册中心，接口应用由 Hertz 框架去替代 HTTP 部分。而后面服务则用 Kitex 将这几个微服务做起来。

**核心业务逻辑：HTTP 服务采用 Hertz 框架，而 RPC 微服务采用 Kitex 框架。**

![tanwan15](/img/users/tanwan/tanwan15.png)


**所用组件**

然后我们可以看一下我们用到的一些组件，例如微服务，必须具备服务注册、发现和配置等功能。

1.  **nacos** **服务注册发现，** **配置中心** **，kitex-contrib/registry-nacos**

我们使用 nacos 作为组件，之前有阿里云的同学过来给我们讲一些分享，他们的销售或是技术支持，都会讲到nacos，因为它在国内使用最广泛、成熟。

同时，它具有现成的代码支持，并且可以与我们的框架结合使用。它也有sdk，但是要跟我们的框架结合起来，还是得用更加现成的，那就是 CloudWeGo 框架自带的 **registry-nacos**，帮我们把这些代码都写好了，只要拿过来用就可以了。

2.  **OpenTelemetry** **链路追踪， kitex-contrib/obs-opentelemetry**

我们讲到微服务框架时，通常会使用链路追踪。我们选择使用 OpenTelemetry，这是框架里边有现成的，而且写得很好。我们在使用过程中也为其做出了一些贡献。

3.  **prometheus** **监控，kitex-contrib/monitor-prometheus**

我们通常做一些服务时候都需要监控，我们使用 Prometheus 进行监控，这些都是在框架中已有的包，拿过来用就可以了，这些都是通过社区共建的方式实现和维护的。

4.  **限流**， **cloudwego** **/** **kitex** **/pkg/limit**

此外，可能会有一些限流。我们之前也有讲到限流，那在微服务里边也是希望它能够限一下流，不会让太多的请求进入，或者说太多的请求过来的时候我们就不处理，能够使我们服务更加坚挺，保证了大部分用户的请求是正常的。

5.  **协议，** **Protobuf** **相对熟悉，** **kitex** **工具好使，支持及时。**

协议的话，我们团队可能更加熟悉的是 **Protobuf**，而 **CloudWeGo** 也很好的兼容此协议，故而我们选用了**Protobuf**。

6.  ...

我们使用的组件基本上是以上这些，还有更多，这里只是做一个展示。这里边有很多是 CloudWeGo 社区共建的，提供了现成的东西供我们参考或直接使用。

![tanwan16](/img/users/tanwan/tanwan16.png)

![tanwan17](/img/users/tanwan/tanwan17.png)


这是一个 settingservice 的监控图，跑了一些 CPU、内存监控，就是运维关注的一般的基础资源的使用情况。


![tanwan18](/img/users/tanwan/tanwan18.png)

我们业务上可能会更加关心延延时，就是下边的图。 **P95、P50 平均值基本上在几十毫秒或者三十**。因为在 K8S 里面， 需要访问我们的一些阿里云服务，比如说数据库， Redis 这些的时候，它从里边到外边也是有一定的时延了。

![tanwan19](/img/users/tanwan/tanwan19.png)

当然，这个 K8S 内外的服务调用是我们无法控制的，但我们控制的是我们业务逻辑代码中使用的时间。因此，这里可能会与我们在ECS上的表现有所不同，我们**需要将那些耗在路上的时间也算进去**。这样，**整体上来看，业务逻辑处理的时间会比这个更低**。

再看下面的图，我们看一下**失败率**，基本上偶尔有一两个跳点，**它的占比** **逼近** **0%** ，就是很低很低。

![tanwan20](/img/users/tanwan/tanwan20.png)

然后我们看一下这个数据量，我们现在用的业务还不是太多，数据量一般，每分钟 100K 上下。我们首次业务实践的把整套基本上跑起来了，那我们把整套跑起来，就把这个框架基本上相对用熟了，我们团队就对框架的掌握也已经算是比较稳妥了。

![tanwan21](/img/users/tanwan/tanwan21.png)

### 游戏 SDK 业务全面重构

以上的实践表现很好，我们想要拓展在更多的业务上使用；前边实践就有点类似 小试牛刀；我们会将不熟悉的一般称为“坑”，并先踩过这些坑，以便在更大的业务场景上安全稳定地使用它们。在更大的业务场景上使用，我们需要把业务拆分得更细。现在，我们已经如下图将服务拆分为多个部分，并将多个业务接进来。可以看到，**Kitex** **有几十个** **RPC** **服务，Hertz 的 HTTP 服务也上十个**。

![tanwan22](/img/users/tanwan/tanwan22.png)

**服务划分**：每个服务是针对单一职责的业务能力封装，专注做好一件事情，可独立开发运行和扩展演化。

1.  **Kitex 微服务 RPC**

-   SettingService
-   AccountService
-   PayService ... 几十个

2.  **Hertz 微服务 HTTP**

-   ApisdkPassport
-   ApiSdkLog
-   WebApi ... 上十个

当然，我们一直在持续推进做这件事。在这两个框架的支撑下，我们可以向其中塞入更多的业务。我们正在不断地向其中塞入更多的业务。如图所示是基本结构：Kitex 集成了一系列的微服务RPC，其中包含业务逻辑重心。对于前端来说，SDK 客户端或其它 HTTP 请求时，使用 Hertz 框架进行简单的接收和RPC调用后端处理逻辑，然后返回。这是我们SDK全面重构的一个实践，我们的团队实践后整体跑得都很稳定。

我们做的一部分事情的呈现就像下面的拓扑图，它是一张展示了我们的服务不断地调用的图。里面有很多服务，形成了一张类似网状的东西。当然，它是有迹可循的，如果一个请求发过来，它调用了几十个服务，那如果你要去排查一些东西，就需要用到链路追踪。

  
![tanwan23](/img/users/tanwan/tanwan23.png)

Trace拓扑图


**Tracing**

微服务架构在对比单架构模式上的性能方面有较大提升，但对应业务复杂度随之上升，出现排查问题难度大，周期长，系统性能瓶颈分析难。 链路追踪是一种用于解决微服务架构中分布式系统调用链路追踪和性能分析的技术。通过在每个服务中加入一些跟踪信息，可以记录请求在各个服务之间的传输路径和所经过的时间，从而形成一个完整的调用链路，协助我们快速定位和解决性能瓶颈问题，进一步优化系统性能。

就像下图，它调用了哪些服务、中间耗的时间是多少、它的实施性是怎样、前置的服务调的是哪一个、在整个链条里边表现是怎样，在下面这个图就能够很清晰的看得到。目前这个图没有任何的不友好的颜色，就是说它没有任何的出错的地方，有出错它也会表现出来。

基本上，看一个报错信息是挺难的，就像刚刚前面那些图，偶尔来一两个跳点，那这个是怎么做出来的？当然，这个图是阿里云的 sls 服务里面做的。但它的底层就是用这个链路追踪，我们现在用的是 CloudWeGo 的 OpenTelemetry，它在 Hertz 和 Kitex 框架里都有提供完整的扩展包，我们把它用起来就可以了。

![tanwan24](/img/users/tanwan/tanwan24.png)

链路追踪

**CloudWeGo** **与社区共建提供了系列组件，可使开发团队快速接入常用的链路追踪**，如我们使用的 kitex-contrib/obs-opentelemetry 和 hertz-contrib/obs-opentelemetry ，提高我们的生产效率，让我们可以集中精力在编写业务相关代码。

社区共建的链路追踪组件对我们很有帮助，**有什么问题就** **针对性加一些追踪** **，也就能看得出问题在哪里**。否则几十个服务如果按照以前的方式，每台机器都去检查，哪怕把日志收集在一起，查到那一个后要把时序一步一步的对应起来，十分折腾。现在有了 CloudWeGo 和社区共建的这组件就很方便了，能够帮我们把事情做好。

**技术支持**

还提供了全面的技术支持 ^_^ 专群对接、业内大神多、 跟进迅速、效果显著...

我们当时选这个框架的时候，CloudWeGo 有一个大群，我们进去提问就有专人对接，最后给我们拉一个专群，里边大神很多。基本上我们的 QPS 可能就几千、几万，CloudWeGo 处理的 QPS可能达到3亿、 10 亿级别，他们面对的场景会比我们更加高并发，更加多用户，更有经验。

CloudWeGo 提供指导是很专业的，处理速度也比较快。当我们上午提出问题，CloudWeGo 当天下午就给出了答案，甚至下午就完成了代码的编写。

沟通落实起来非常高效，有时甚至比我们的同事更加高效，效果也很明显。下方左图可见，我们当时提出的问题很快就得到了审查并快速处理了。可见“opentelemetry”后边的数字是2，意味着是早期提出的东西，就很快地解决了。再次感谢 CloudWeGo 给予的大力支持。

![tanwan25](/img/users/tanwan/tanwan25.png)

## 运维之 SLB 后端 Pod 不优雅下线

刚刚介绍了这一整套系统的落地过程。在落地过程中，可能会出现一些小问题，但这些问题是技术工作中常有的，不会影响整个系统大局的正常运行。先看下图，在上线过程中，我们可以进行压测，发现当滚动更新时，它就不优雅地下线，导致一些丢包。处理排查时，我们会发现它确实是一些设置上的问题。如果这个进程还在里面，我们可以让它延迟一下，因为它一直在提供服务，即使有请求过来，它也可以继续提供服务。

SLB 后端被移除的瞬间，客户端压测直接通过 SLB 的 IP 访问服务发生超时。经过团队协作抓包定位，发现超时的请求回复报文没有到达SLB后端，属于 SLB 后端不优雅下线情况下的丢包场景。

![tanwan26](/img/users/tanwan/tanwan26.png)

> 大家有兴趣的话可以参考一下阿里云的链接，如果有相似场景可以留意一下。
>
> 参考链接： https://help.aliyun.com/document_detail/86531.html

### 运维之滚动更新信号量处理

接着，我们再讲一个运维上的问题，就是在滚动更新时，有一个信号量的处理。我们期望在k8s滚动更新新版本时，让它优雅关闭，就是走到 graceful shutdown 而非 force exit 。在排查和调试时，我们会发现它走到了我不期望它走到的地方。

可能 K8S 与我们平常使用的 ECS 系统有所不同，因为在 K8S 中，信号量在发送终止指令或滚动更新时可能会不同。那么，我们希望程序能够正常处理这些情况。为了解决这个问题，我们修改了处理函数，并将其提交给了 CloudWeGo 团队。他们为我们提供了一个自定义函数，并将其集成到 PR 中。最后，该 PR 是Hertz0.2.0 发布。下边这个图就是我们当时的一些处理过程。

> Hertz v0.2.0 ：https://github.com/cloudwego/hertz/pull/143

![tanwan27](/img/users/tanwan/tanwan27.png)

### 取 Oss 文件效率问题的暴露

接下来分享我们实际在代码层面的一些优化，简单讲解优化事项。

抽一个事项来说，就是我们一开始这个服务上线的时候，能看得到

1.  基本要170ms左右，哪怕是内部调用也很慢
1.  在内网测试环境一直没测出问题

我们使用链路追踪，分段细化加入 span 记录链路时间，分析每个 span 的耗时，找出耗时过长的 span，然后查看线上相关数据。

-   **找出问题：**

1.  发现耗时都是在 SettingService 读取 oss 文件时
1.  猜测并发时自身的 dataMap 经常有锁住
1.  读取非必需的文件时，数据经常为空，具有缓存穿透问题
1.  过多的缓存冷数据

![tanwan28](/img/users/tanwan/tanwan28.png)


-   **问题优化**

1.  初步使用 gcache 优化，阻止多个协程并发读取同一文件
1.  根据 LRU 算法，区分冷热数据，按量使用 Goroutine 定时主动并行更新热点数据
1.  对于非必需配置文件，进行缓存穿透处理，缓存 nil 值
1.  加强 func 的单一职责，减少读取文件的数量

**优化效果: 平均时延降低到10+ms**

我们确实是几十毫秒就搞定了，但是那个图是之前的，现在我们的业务能够达到 10 毫秒左右。

![tanwan29](/img/users/tanwan/tanwan29.png)

这就是我们使用 CloudWeGo 架构的落地，从前边的这个一小点突破，到这个框架熟悉了后选择了一个很重要的业务场景去用它，然后我们的开发人员都熟悉了后在更大的业务场景下使用 CloudWeGo 重构了我们整个 SDK 接口。

第三个问题是我们在运维或技术代码方面存在一些小问题。因此，我们决定将整个流程走下来，整体运行得挺好的。

### 整体收益

1.  **稳定性提升**

-   **上限提升** **压测** **单** **Pod** **1c2g 400+** **qps** **。** 我们追求的是稳定性，即实现方式的上限提升，从而单台机器处理能力也更高了。现在，我们单台单个 Pod 可以测试出1核 2G 的内存，基本上可以测试到 400 个 QPS。在登录场景下，明显比php-fpm的处理效果要好。
-   **自动伸缩容。** 第二个是 K8S 这一块的自动伸缩容，这一块是 K8S 天生以来的好东西。它请求量大了，会自动扩多个Pod并调度起来，从而可承载的业务请求量更大。
-   **可承载的业务请求量更大**

2.  **CloudWeGo** **提高开发成效**

整体收益是使用 CloudWeGo 框架后，我们提高了开发效率、提升了性能可靠性、简化了部署流程，现在一键部署很方便。此外，我们还增加了业务弹性，可以往里面塞更多东西，从而降低成本，符合当前降本增效的主题，但需要不断地往里边塞更多业务。

-   提升性能和可靠性
-   提高开发效率
-   简化部署和管理
-   增加业务弹性
-   降低IT成本
-   ......

3.  **团队能力提升**

我们的人员已经具备一定的技术水平，他们确实需要扩充自己的技术栈，丰富团队的技术能力。我们学习了 Go 语言，并与 CloudWeGo 一起成长，我们自己才能在行业抢占先机。我们的定位不同，所以最终的结果也会不同。我们整个团队可以应对更广泛的业务场景，因为以前可能只能处理 a 类业务，现在我们团队也可以处理b、c、d、e类业务。

-   丰富队伍的技术栈
-   强者 CloudWeGo 指引，自身更强
-   可应对更宽广的业务场景
-   ...

因此，我们用了 CloudWeGo 之后，整体收益还是比较不错的。

## 给相似业务的参考建议

1.  **根据团队水平、确定团队推进的节奏。**

有一些团队是很高水平的，因此他们可能就直接使用了这些技术，有些团队可能已经成功转型了。但是我们的团队可能还没有完全掌握这些技术，因此我们需要逐步推进，不断突破和运用，团队用熟之后，再整体扩大。类似要试点，试点完之后再扩大，控制好这种节奏。

2.  **做好相关业务拆分的粒度。**

业务拆分看各自团队的各自场景，像我们贪玩游戏，基本上我们能想得到的一般是拆服务。主要是用户相关服务拆一个服务、支付相关服务拆一个服务，当然你的配置也可以拆一个服务，当然还有更多，甚至于你的弹窗服务也可以拆一个服务。

像我们前一阵子也了解到字节跳动有好几万个服务，我们现在也才上百个服务，所以以后要做的事情还有很多，我们的业务也在不断地发展了。然后这个服务拆分这部分，根据各自的业务团队来，根据业务场景来自己 hold 住就比较 OK 了。

3.  **运维团队的专业水平、学习速度，善用外部资源。**

运维这一块，基本上我们写好代码，也要部署上去。如果手动的话，确实运维团队也要不断地学习。比如说你以前可能是在服务器上操作的，现在可能都在后台界面，不断地去使用新的工具，要不断地提升自己的学习速度、善用外部资源。

如果你不太熟，可以找别的人来教一下你。比如说云服务提供商，他要卖一个产品，肯定把这个产品用熟了之后才卖给你，那你跟他学到的东西就很多了。比如，我们和 CloudWeGo 就学到了很多，我们讲站在巨人的肩膀上，自己也可以看得更高，摸到的天花板就更高。

4.  **技术业务表现的数据监控 及 相关工具的使用。(链路追踪)**

在做实践落地的这个过程中，一定要做好这个数据监控，要用好相关工具。我们现在用到了比如说链路追踪，这个工具的体现就很好。

因为每个团队不一样，每个场景不一样，每个行业不一样，大家就按照自己的场景来做一下自己的参考。
