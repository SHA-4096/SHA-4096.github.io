---
title: 微服务框架——从gRPC到Service Mesh
date: 2024/5/17
---


>微服务 (Microservices) 是一种软件架构风格，它是以专注于单一责任与功能的小型功能区块 (Small Building Blocks) 为基础，利用模块化的方式组合出复杂的大型应用程序，各功能区块使用与**语言无关** (Language-Independent/Language agnostic) 的 API 集相互通信。
>——wikipedia

随着分布式系统的兴起，人们不再满足于将一个系统的不同应用进行分布式部署，而是将一个应用拆分成一个个的微服务，将它们部署到不同的主机上。微服务的优点有很多，比如能够简化开发、逻辑清晰、扩展性强等等。但是同时，微服务也面临复杂度高、运维复杂等问题。为了使微服务更加好用，一系列的微服务框架不断地迭代更新，这个过程中出现了许多有趣的架构设计。本文从gRPC、注册中心、dubbo等微服务相关的技术入手介绍微服务中的一些概念和架构

## 前置知识——RPC
*RPC*是远程过程调用（Remote Procedure Call）的简写。 RPC 的主要功能目标是让构建分布式计算（应用）更容易，在提供强大的远程调用能力时不损失本地调用的语义简洁性。为实现该目标，RPC 框架需提供一种透明调用机制，让使用者不必显式的区分本地调用和远程调用。

## 从最简单的gRPC聊起
传统的RPC将接口定义写在业务逻辑里，这导致服务端和客户端之间的协调变得相当麻烦。为此，谷歌将接口的定义部分写到了protobuf IDL(Interface Definition Language) 里，然后通过[代码生成工具]([protocolbuffers/protobuf: Protocol Buffers - Google's data interchange format (github.com)](https://github.com/protocolbuffers/protobuf))生成对应的客户端和服务端代码。然后客户端就可以通过`服务端地址+参数`这一组信息，通过接口函数向服务端发起调用，看起来一切都很好
![](https://grpc.io/img/landing-2.svg)
**但是它真的很好吗**

设想一下，假设我们服务主机的IP地址发生了改变，在客户端所有代码和配置都不改变的情况下，gRPC还能正常工作吗？ 显然不能，因为gRPC的调用是和服务提供方的地址绑定的，只要服务端的地址改变，就需要客户端作出相应的调整

简单的gRPC还有另一个问题。设想一下，如果服务端和客户端是由不同的开发者开发的，那么对于客户端的开发者而言，一个服务的名称和参数等信息都必须通过查看源代码或接口文档获取，一个服务是否处于可用状态更是不可知的，这一看就不是特别的高可用

于是，我们希望有这样一种东西：一个能够告诉我们当前能够访问的微服务有哪些以及它们如何调用的“注册中心”， 它的地址相对固定，能够维护可用的微服务的一组信息，并能够被所有的客户端访问，为客户端提供这些服务的地址、名称、参数等。（实际上，gRPC中，服务端本身就可以视为自身的注册中心，只不过这个注册中心没有服务发现的功能）

## 注册中心——以Nacos为例

注册中心是微服务架构中的纽带，类似于“通讯录”，它记录了服务和服务地址的映射关系。在分布式架构中，服务会注册到这里，当服务需要调用其它服务时，就到这里找到服务的地址并进行调用。注册中心本质上是为了解耦服务提供者和服务消费者。

`Nacos /nɑ:kəʊs/` 是 Dynamic Naming and Configuration Service的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Nacos 提供了一组简单易用的特性集，以快速实现动态服务发现、服务配置、服务元数据及流量管理。
![](https://cdn.nlark.com/yuque/0/2019/jpeg/338441/1561217892717-1418fb9b-7faa-4324-87b9-f1740329f564.jpeg)
图中各个实体都是通过网络通信的，核心机制由`Nacos Server`实现。 `Provider APP`向`Nacos Server`提出`服务注册`的请求， 然后通过服务发现， `Consumer APP`可以从`Nacos Server`服务的信息。此时，从`Consumer APP`的视角来看，它只需要记住`Nacos Server`的地址，就可以实现微服务信息的获取和调用，比起为每个`Provider APP`单独维护地址，简直方便了太多

*下面通过例子直观感受一下how it works*

首先，可以用docker超快地在本地启动一个Nacos（这里用最简单的单机模式）
```bash
git clone https://github.com/nacos-group/nacos-docker.git
cd nacos-docker
docker-compose -f example/standalone-derby.yaml up
```

然后，我们就可以用`curl`和Nacos Server交互啦

比如我们要注册一个服务，只需要发送如下请求：
```bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
#
```
url的含义是，我们有一个叫`nacos.naming.Service`的服务，它跑在ip为20.18.7.10的主机的8080端口上

而想要查询的话：
```bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
```
这条命令返回的数据是这样的：
```json
{
  "name": "DEFAULT_GROUP@@nacos.naming.serviceName",
  "groupName": "DEFAULT_GROUP",
  "clusters": "",
  "cacheMillis": 10000,
  "hosts": [
    {
      "instanceId": "20.18.7.10#8080#DEFAULT#DEFAULT_GROUP@@nacos.naming.serviceName",
      "ip": "20.18.7.10",
      "port": 8080,
      "weight": 1,
      "healthy": true,
      "enabled": true,
      "ephemeral": true,
      "clusterName": "DEFAULT",
      "serviceName": "DEFAULT_GROUP@@nacos.naming.serviceName",
      "metadata": {},
      "instanceHeartBeatInterval": 5000,
      "instanceHeartBeatTimeOut": 15000,
      "ipDeleteTimeout": 30000,
      "instanceIdGenerator": "simple"
    }
  ],
  "lastRefTime": 1715930453232,
  "checksum": "",
  "allIPs": false,
  "reachProtectionThreshold": false,
  "valid": true
}
```
它包含了名字为`nacos.naming.serviceName`的微服务的各类信息，不难想到，只要`ComsumerAPP`获取到了这些信息，他就可以直接通过网络向这个服务对应的主机发起调用请求。许多这样的Comsumer,Provider,Nacos Server组合起来，就变成了一个所谓的`服务网格`

**还存在什么问题呢？**

安全性问题。如果服务网格内有一些提供敏感信息的服务（这类应用还是很多的），要怎么保证这个服务提供的信息不泄露呢？这时就需要鉴权和加密机制。

为了实现高可用，一个服务可能会有多个副本，那么将请求合理地发送到这些副本则又涉及到了负载均衡……这些能力都是单纯的注册中心无法提供的（实际上，注册中心有点像是一个功能比较完备的，使用网络请求操作的键值数据库，这些能力自然也不应该让注册中心来承担）。

除此之外，一个服务的运行是否正常，服务日志的查看等，似乎都不是注册中心的职责，为此，开发者可能还要额外做许多与业务无关的工作

## 以Sidecar为基础的ServiceMesh——以Istio为例

Istio 是一个开源服务网格，它透明地分层到现有的分布式应用程序上。Istio 强大的特性提供了一种统一和更有效的方式来保护、连接和监视服务。 Istio 是实现负载均衡、服务到服务身份验证和监视的路径——只需要很少或不需要更改服务代码。它强大的控制平面带来了重要的特点：
- 使用 TLS 加密、强身份认证和授权的集群内服务到服务的安全通信
- 自动负载均衡的 HTTP、gRPC、WebSocket 和 TCP 流量
- 通过丰富的路由规则、重试、故障转移和故障注入对流量行为进行细粒度控制
- 一个可插入的策略层和配置 API，支持访问控制、速率限制和配额
- 对集群内的所有流量（包括集群入口和出口）进行自动度量、日志和跟踪

![](https://istio.io/latest/zh/docs/ops/deployment/architecture/arch.svg)


Istio 服务网格从逻辑上分为**数据平面**和**控制平面** 。

- **数据平面** 由一组被部署为 Sidecar 的智能代理（[Envoy](https://www.envoyproxy.io/)） 组成。这些代理负责协调和控制微服务之间的所有网络通信。 它们还收集和报告所有网格流量的遥测数据。
    
- **控制平面** 管理并配置代理来进行流量路由。

下面介绍Istio的几个基本组件
### Envoy
也就是上面的Proxy部分
Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。
Envoy通常是一个独立的容器，这个容器和业务逻辑跑在同一个kubernetes pod里，这种结构下的Envoy被称为**sidecar**
### Istiod
- Istiod 提供服务发现、配置和证书管理。
- Istiod 将控制流量行为的高级路由规则转换为 Envoy 特定的配置， 并在运行时将其传播给 Sidecar。Pilot 提取了特定平台的服务发现机制， 并将其综合为一种所有符合 [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api) 的 Sidecar 都可以使用的标准格式。
- Istio 可以支持发现 Kubernetes 或 VM 等多种环境。
- Istiod [安全](https://istio.io/latest/zh/docs/concepts/security/)通过内置的身份和凭证管理， 实现了强大的服务对服务和终端用户认证。可以使用 Istio 来升级服务网格中未加密的流量。 使用 Istio，运营商可以基于服务身份而不是相对不稳定的第 3 层或第 4 层网络标识符来执行策略。 此外，可以使用 [Istio 的授权功能](https://istio.io/latest/zh/docs/concepts/security/#authorization)控制谁可以访问特定的服务。
- Istiod 充当证书授权机构（CA）并生成证书，以允许在数据平面中进行安全的 mTLS 通信。

所有微服务的通信都需要被Envoy代理，数据平面和控制平面的南北向通信，以及数据平面之间的东西向通信，**都不需要微服务的业务逻辑直接参与**，这就实现了业务逻辑和通信逻辑的解耦。最后的服务网格看起来会像这样：
![](https://pic4.zhimg.com/80/v2-8a9cc161a34d97f36ead06d0abc5b1fb_1440w.webp)

其中蓝色的是Envoy，绿色的是业务逻辑

## 基于SDK的ServiceMesh——以Apache Dubbo为例

尽管Istio的架构看起来非常完美，但是有一点是无法忽视的，就是大量Envoy带来的额外资源开销。有时候我们根本用不上Envoy提供的一系列高级功能，但是它们仍旧作为服务跑在每一个sidecar里，占用着系统的资源

而Apache Dubbo的解决方案是，直接将Sidecar打包成SDK，嵌入到业务代码的底层，并为业务逻辑提供一系列易于使用的接口。这样一来，通信由SDK部分的代码完成，对于业务逻辑而言，它也不需要关心网络通信是如何实现的。
*换言之，SDK的接口函数取代了原本业务代码和sidecar通信的地位，而SDK本身就像是一个嵌入到了业务逻辑代码的Sidecar*
![](attachments/Pasted%20image%2020240517093232.png)

dubbo部署后的结构同样分为控制面和数据面。与Istio不同的是，因为dubbo SDK本身是语言相关的，因此其有一套开发框架，来减弱乃至消除这种语言的相关性

### 数据面
从数据面视角，Dubbo 帮助解决了微服务实践中的以下问题：

- Dubbo 作为 **服务开发框架** 约束了微服务定义、开发与调用的规范，定义了服务治理流程及适配模式
- Dubbo 作为 **RPC 通信协议实现** 解决服务间数据传输的编解码问题
![](https://cn.dubbo.apache.org/imgs/v3/what/framework1.png)

**针对不同语言业务逻辑的服务开发框架**
![](https://cn.dubbo.apache.org/imgs/v3/what/framework2.png)
>微服务的目标是构建足够小的、自包含的、独立演进的、可以随时部署运行的分布式应用程序，几乎每个语言都有类似的应用开发框架来帮助开发者快速构建此类微服务应用，比如 Java 微服务体系的 Spring Boot，它帮 Java 微服务开发者以最少的配置、最轻量的方式快速开发、打包、部署与运行应用。
>微服务的分布式特性，使得应用间的依赖、网络交互、数据传输变得更频繁，因此不同的**应用需要定义、暴露或调用 RPC 服务，那么这些 RPC 服务如何定义、如何与应用开发框架结合、服务调用行为如何控制？这就是 Dubbo 服务开发框架的含义，Dubbo 在微服务应用开发框架之上抽象了一套 RPC 服务定义、暴露、调用与治理的编程范式**，比如 Dubbo Java 作为服务开发框架，当运行在 Spring 体系时就是构建在 Spring Boot 应用开发框架之上的微服务开发框架，并在此之上抽象了一套 RPC 服务定义、暴露、调用与治理的编程范式。

具体地，dubbo通过`protobuf IDL`定义服务的接口（就像gRPC开发规范中那样），但是在其中加入许多额外的字段，来标识其在不同语言中的对应信息（比如这个接口在java和golang中的接口函数分别应该叫什么，这个接口使用什么通信协议等）；然后，通过不同语言的代码生成工具，将`IDL`**变成对应语言的逻辑代码，并提供可以调用的接口函数**

例如，一个典型的java和golang通用的protobuf文件可能是这样的：
```protobuf
syntax = "proto3";
package org.apache.dubbo.sample;

option go_package = "/proto;proto";
//package of go
option java_package = 'org.apache.dubbo.sample';
option java_multiple_files = true;
option java_outer_classname = "HelloWorldProto";
option objc_class_prefix = "WH";

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello(HelloRequest) returns (HelloReply);
  // Sends a greeting via stream
  //  rpc SayHelloStream (stream HelloRequest) returns (stream HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
### 服务治理（抽象的控制面）
服务开发框架解决了开发与通信的问题，但在微服务集群环境下，我们仍需要解决无状态服务节点动态变化、外部化配置、日志跟踪、可观测性、流量管理、高可用性、数据一致性等一系列问题，我们将这些问题统称为服务治理。

Dubbo 抽象了一套微服务治理模式并发布了对应的官方实现，服务治理可帮助简化微服务开发与运维，让开发者更专注在微服务业务本身。控制面与数据面的通信也是通过Dubbo SDK“代理”的
![](https://cn.dubbo.apache.org/imgs/v3/what/governance.png)
为了实现服务治理，就必然还是要使用Nacos等，它们是通过Dubbo Admin提供的控制台统一组织的

Admin 控制台提供了 Dubbo 集群的可视化视图，通过 Admin 你可以完成集群的几乎所有管控工作。
- 查询服务、应用或机器状态
- 创建项目、服务测试、文档管理等
- 查看集群实时流量、定位异常问题等
- 流量比例分发、参数路由等流量管控规则下发

### 服务网格
有趣的是，Dubbo和Istio并不是完全平行的关系。dubbo本身就可以作为Istio数据面的一部分，甚至可以作为Istio集群内的一个服务……这也就意味着Dubbo有着极高的灵活性
![](https://cn.dubbo.apache.org/imgs/v3/mesh/mix-mesh.png)

# 总结

从最早的RPC，到gRPC，再到注册中心、Istio以及Dubbo，微服务框架的发展路径是在不断将底层细节与业务逻辑解耦的同时简化服务治理、提高可用性的过程。

# 参考
[Introduction to gRPC | gRPC](https://grpc.io/docs/what-is-grpc/introduction/)  
[Nacos 快速开始](https://nacos.io/zh-cn/docs/quick-start.html)  
[Istio / 架构](https://istio.io/latest/zh/docs/ops/deployment/architecture/)  
[什么是 Service Mesh - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/61901608)  
[了解 Dubbo 核心概念和架构 | Apache Dubbo](https://cn.dubbo.apache.org/zh-cn/overview/what/overview/)  
[apache/dubbo-go-samples: Apache dubbo (github.com)](https://github.com/apache/dubbo-go-samples) 