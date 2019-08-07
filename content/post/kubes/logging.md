---
title: "kubernetes 日志"
date: 2019-07-24T16:32:50+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---

容器部署和进程部署应用的日志采集方式有所不同，在使用容器之前一般是通过 supervisor 的方式管理进程，supervisor 会把 stdout、stderr 的日志打到相应的目录，或者应用把日志写到节点的某个指定的目录。以 docker 为例，容器被 docker daemon 管理，docker daemon 相当于容器的父进程，docker daemon 可以拿到应用的 stdout、stderr，docker 会把日志写到指定的目录下，docker 支持[多种日志驱动](https://docs.docker.com/config/containers/logging/configure/)，k8s 默认使用 json-file 的方式。<!--more-->
```golang
func main() {
	cmd := exec.Command("ls", "-lah")
	out, err := cmd.CombinedOutput()
	if err != nil {
		log.Fatalf("cmd.Run() failed with %s\n", err)
	}
	fmt.Printf("combined out:\n%s\n", string(out))
}
```

## 日志类型

1. 系统组件的日志，apiserver、controller-manager、kubelet 等，kubelet 比较特殊，木有做容器化，日志还是通过 supervisor 的方式，所有容器化了的日志都会被输出 /var/log/... 目录下
查看 kubelet 的日志
```bash
journalctl -l -u kubelet
```
2. 应用组件的日志，应用组件的日志通常会被输出到文件或者终端，使用容器部署应用推荐应用方把日志输出到终端，因为 pod 会漂移，采集容器中的文件日志需要一些额外的开销。

## 日志采集方案

在体量较小的情况下，日志通常就是用来排查问题，在体量变大的情况下，日志统一管理和收集会变得尤为重要，除了安全审计、排查问题之外，很多公司通过对日志进行分析，也可以挖掘出这些日志数据的价值。

k8s 推荐的方式是 agent、es、kibana 来收集检索日志，agent 一般使用 fluentd，通过 daemonset 的方式部署 fluentd，保证一个节点上运行一个 fluented，然后 agent 在节点上采集日志发送后端的 backend 持久化存储日志信息。

比较特殊的情况是如何采集文件类型的日志，通常不推荐日志输出到文件中，如果要做的话，你可以把日志输出到一个 empty dir volume 下，和你的日志采集工具约定好往这个指定的 empty dir 下采集。也可以跑一个 sidecar 容器和你的应用 pod 放在一起，sidecar 把应用容器中文件 tail 到终端来采集，两种方式对应用本身都是一点侵入性，所有还是推荐把日志打到终端即可。

## reference

[docker log drive](https://docs.docker.com/config/containers/logging/configure/)

[Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
