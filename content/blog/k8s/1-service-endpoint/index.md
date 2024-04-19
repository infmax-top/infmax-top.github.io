---
title: ":calendar: 20240422 K8S Service和Endpoint"
date: 2024-04-22T19:24:21+08:00
description: ""
tags: ["云原生", "k8s", "网络", "分布式调度"]
categories: ["分布式系统"]
draft: true
---

在云原生K8s环境下, 应用以pod模式部署, 不再是固定的机器部署. 应用随时可能因为发布或者故障转移等情况而发生Pod的重建, 这带来Pod IP的变化, 因此, 应用的服务发现成为了重要的一环.  K8s 通过service的方式, 为应用提供了统一且固定的接入点.

Service是一种抽象的机制，用于暴露一组Pod的稳定的IP地址和端口。它提供了一个单一的入口点，使得其他的应用程序可以通过这个入口点访问这些Pod，从而实现了Pod的负载均衡和服务发现。

# Service用法
Service可以通过多种方式将请求路由到后端的Pod，例如：

- ClusterIP：在集群内部暴露Service，只能在集群内部访问。
- NodePort：在每个节点上暴露Service，可以通过节点的IP地址和NodePort访问Service。
- LoadBalancer：使用云服务提供商的负载均衡器来分配外部流量到Service。
- ExternalName：将Service映射到一个外部地址。
- Service还支持选择器（selector）机制，即通过标签（label）来选择一组具有相同标签的Pod。当一个Service被创建时，可以指定一个或多个标签，Service会自动将具有这些标签的Pod组成一个虚拟的服务。当Pod的标签发生变化时，Service会自动更新它所代表的Pod集合。


# ClusterIP/NodeIP/PodIP的关系
在理解Serivce的过程中, 我们肯定在Service的描述中看到过ClusterIP. 那么ClusterIP, PodIP, NodeIP 三者的关系是什么?


关于ClusterIP, 这里需要了解下什么是 [VIP(Virtual IP](../2-network-vip)



# Endpoints 和 Service的关系
{{< alert >}}
Service的名字和Endpoints的名字要一样
{{< /alert >}}