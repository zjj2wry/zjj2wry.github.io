---
title: "kubernetes 中的乐观并发控制"
date: 2019-03-14T14:33:25+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---

## 乐观并发控制

乐观并发控制指的是它保持乐观的态度，认为并发执行过程不会对共享数据出现竞争问题。只在修改数据的时候检查共享数据是否出现竞争问题，如果在修改的时候其他的进程没有修改过该共享数据，则修改成功。<!--more--> 

### kubernetes 中的乐观并发控制(compare and swap)

k8s 的资源都有一个 metadata.ResourceVersion 字段，在更新一个资源的时候都需要带上该字段，也就是你在执行 update 操作之前通常会先执行 get 操作，server 端会比较该字段，如果相同则更新成功，更新成功后会修改该字段，如果有另外一个进程或者线程修改该字段则会失败。

### kubernetes 中的乐观并发控制的高级使用

#### leader election

leader election 指的是一个程序为了高可用会有多个副本，但是每一个时候只允许一个进程在工作，比如说转账业务。再比如 k8s 的 controller manager, 因为 controller manager 是 k8s 的核心组建之一，需要保证高可用，假设你是一个多 master 的集群，挂了一个节点应该保证你的集群可以正常的工作，但是 controller manager 中的 controller 其实做的事情是一样的，大部分事情就是 watch 资源，更新 status，上面说过多个进程同时更新一个资源的时候会失败，controller manager 多个副本同时工作没有什么意义。k8s 基于乐观并发控制实现了 leader election。
```bash
✗ kubectl get ep -n kube-system kube-scheduler -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"kube-master-1_ad5220de-2442-11e9-91f6-52540025e0cf","leaseDurationSeconds":15,"acquireTime":"2019-02-22T02:09:15Z","renewTime":"2019-03-14T07:37:19Z","leaderTransitions":1}'
  name: kube-scheduler
  namespace: kube-system
```
holderIdentity：表示当前那个副本在工作
leaseDurationSeconds：租期的时间
acquireTime：获得锁的时间
renewTime：刷新锁的时间
leaderTransitions：leader 交换的次数

大概的逻辑就是刚启动的时候多个副本会去同时更新 kube-scheduler 这个 endpoint(获取锁)，这个时候会有一个副本更新成功，其他的会更新失败, 然后这个锁会有一个租期，这个进程会有个时间间隔(小于租期，一般设置是租期间隔的一般)不断的去更新 kube-scheduler 这个 endpoint(获取锁), 其他的进程也会去尝试获取锁，如果 leader 所在的节点挂了，那么其他的进程就会得到锁成为 leader。

#### 抢占 resource quota 资源
```yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: ResourceQuota
  status:
    hard:
      requests.nvidia.com/gpu: "1"
    used:
      requests.nvidia.com/gpu: "0"
```

k8s 的分区有个 quota 机制，可以用来限制分区可以使用的资源大小，比如上图中某分区的 quota 中 gpu 可用数量是 1，如果有两个 pod 要使用 gpu，那么会有一个 pod 无法创建成功。大概的逻辑是创建 pod 的时候会先去检查 quota 的资源够不够(hard-used)，如果够的话，会更新 quota 的 status，因为同时更新会有一个失败，然后资源就被其中一个使用了，另一个永远无法成功。但是现在的 resource quota 还是有问题，可能会出现更新了 quota 的 status 成功了，但是 pod 创建失败的情况，然后 status 的资源一直被占用。现在 kubernetes 是通过在 quota controller 中设置了 5 mins 的 resync(重新把 cache 里的资源 enqueue) 时间解决这个问题。

## 悲观并发控制
悲观并发控制保持着悲观的态度，认为并发执行过程一定会出现问题。

传统的锁机制都是悲观并发控制的实现，通过加锁从而保证多线程互斥地修改和访问共享数据，那么多线程就不会竞争使用共享数据，也就不会产生线程安全问题。

读锁(共享锁): 对该共享数据加锁，可以读取和修改，加锁期间其他的进程不能读写，也不能对该共享数据加锁
写锁(排它锁): 对该共享数据加锁，可以读取，加锁期间其他的进程只能加读锁。
