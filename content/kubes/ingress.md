---
title: "kubernetes ingress"
date: 2019-07-24T18:14:30+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---

kubernetes ingress 的作用是为你的服务提供服务的入口让其他人访问，k8s 支持一下几种方式<!--more-->
    
    1. nodePort 方式，服务端口固定 30000~32XXX，LB 指向集群中的任意几台。所有的节点都会自动为每个服务创建 ipvs 规则，只有 LB 绑定的节点规则有意义。nodePort 是 k8s 自动分配的，使用这种方式要手动设置端口避免冲突。只能用于 debug，流量要多做一次 nat。

        lb --> node（ipvs nat）--> nginx pod



    2. LB 绑定到固定的节点上，在这些节点上运行 ingress conroller， 要保证 k8s 调度 pod 到这个节点上，且只能在节点上运行一个副本，因为 pod 要共享节点网络，占用宿主机上的 80/443 端口，所以一个节点上只能跑一个 nginx，要创建多个集群内 lb 只能在不同的节点上创建，自己要去维护 lb 和节点的关系。流量和节点网络相关。

        lb --> node(deamonset + hostnetwork，相当于在节点上跑了一个 nginx)



    3. LB 类型的 service，LB 直接绑定 nginx pod endpoint，80/443 端口使用的是 pod 自身的端口，可以任意创建 nginx pod 做内网的 lb，不存在端口冲突的情况。流量和 pod 的网络相关。

        lb --> nginx pod endpoint
       
      
私有云一般都是第 2 种方式，公有云建议使用第 3 种方式，可以一个服务一个 lb。       
