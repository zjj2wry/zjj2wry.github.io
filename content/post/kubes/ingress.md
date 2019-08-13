---
title: "kubernetes ingress 实践"
date: 2019-07-24T18:14:30+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---
## concept
[如何为 Service Mesh 选择入口网关](https://mp.weixin.qq.com/s/UVFhKR3JWp87WKnRa6de1Q)

[A Deep Dive into Kubernetes External Traffic Policies](https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies)

[追踪 kubernetes 服务的SNAT](https://ieevee.com/tech/2017/09/18/k8s-svc-src.html)

## ingress 部署方式
1.nodePort 方式，服务端口固定 30000~32XXX，LB 指向集群中的任意几台。所有的节点都会自动为每个服务创建 ipvs 规则，只有 LB 绑定的节点规则有意义。nodePort 是 k8s 自动分配的，使用这种方式要手动设置端口避免冲突。需要手动配置端口映射关系。

```bash
LB --> NODE（ipvs nat）--> nginx pod endpoint
```

2.LB 绑定到固定的节点上，在这些节点上运行 ingress conroller， 要保证 k8s 调度 pod 到这个节点上，且只能在节点上运行一个副本，因为 pod 要共享节点网络，占用宿主机上的 80/443 端口，所以一个节点上只能跑一个 nginx，要创建多个集群内 lb 只能在不同的节点上创建，自己要去维护 lb 和节点的关系。流量和节点网络相关。

```bash
LB --> NODE(deamonset + hostnetwork，相当于在节点上跑了一个 nginx)
```

3.LB 类型的 service，会在阿里云的后台创建四层的 SLB， 自动映射 80/443 端口到 ingress controller 的 nodeport。

```bash
LB → NODE(ipvs nat) -->nginx pod endpoint
```

结论：

nodePort 和 LB 类型的 service 最后流量是一样的，LB 类型的 service 不需要手动管理端口映射关系和端口的分配问题。会自动绑定节点到 LB。如果是公有云环境建议使用第3种，私有云使用第一种或者第二种。第二种需要考虑升级对服务的影响，因为要使用 daemonset 部署，要先删后创建。

## ingress 配置优化
选择 ingress 运行的节点
```bash
kubectl label nodes cn-beijing.172.16.132.150 node-role.kubernetes.io/ingress="true"
```

将 ingress 调度到指定的节点上
```bash
kubectl -n kube-system patch deployment nginx-ingress-controller -p '{"spec": {"template": {"spec": {"nodeSelector": {"node-role.kubernetes.io/ingress": "true"}}}}}'
```

通过podAntiAffinity 避免pod 流量不均衡

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: name
              operator: In
              values:
              - frontend
          topologyKey: kubernetes.io/hostname
```

副本数增加至 3

```bash
kubectl -n kube-system scale --replicas=3 deployment nginx-ingress-controller
```

(option) 修改 externalTrafficPolicy 为 Local，保留源 IP，减少 node 之间额外的一跳，local 会将 ingress pod 所在的节点加到 ali slb，Cluster 会将集群中的节点加到 ali slb。
```bash
kubectl -n kube-system patch svc nginx-ingress-lb -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

设置 ingress controller pod 的 request.cpu (对应 croup cpushares，按照权重分配，如果不设置默认为 2)，设置request.cpu 避免有服务 cpu 使用过高，导致 ingress controller 能分配到的 cpu 太少
```
kubectl set resources deployment --requests=cpu=4 nginx-ingress-controller -n kube-system
```

设置 ingress controller pod 为 critical pod，避免节点内存压力被驱逐
```
kubectl -n kube-system patch deployment nginx-ingress-controller -p '{"spec": {"template": {"spec": {"priorityClassName": "system-cluster-critical"}}}}'
```

(base on needs) 修改 lb 的类型为内网 slb (目前集群不会直接对外暴露服务，实际操作似乎需要先改成 nodeport，让公网的被删除，再改成 loadbalancer 类型后加上 annotation)
```bash
kubectl annotate svc nginx-ingress-lb -n kube-system service.beta.kubernetes.io/alicloud-loadbalancer-address-type="intranet"
```

(option) 添加 taints 和 tolerations 让 ingress 独占节点。

(would be better) custom metrics 扩容 ingress controller 