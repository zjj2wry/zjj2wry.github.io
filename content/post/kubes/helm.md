---
title: "helm 安装常见的 issue"
date: 2019-07-25T19:28:47+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---

现在 K8S 应用基本上都是使用 helm 作为包管理，想要部署一些应用都会使用到 helm，由于 helm 存在 server 端 tiller，客户端的权限并不能决定你能做什么，即使你本地的 kubeconfig 文件
是 admin 权限，你能做的事情也被 tiller 限制，第二个是客户端和 server 版本的兼容性问题。<!--more-->

## 客户端和 server 端口版本不一致

```bash
➜  deployment git:(master) ✗ helm install stable/node-problem-detector --name node-problem-detector
Error: incompatible versions client[v2.14.2] server[v2.7.0]
➜  deployment git:(master) ✗ helm version
Client: &version.Version{SemVer:"v2.14.2", GitCommit:"a8b13cc5ab6a7dbef0a58f5061bcc7c0c61598e7", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.7.0", GitCommit:"08c1144f5eb3e3b636d9775617287cc26e53dba4", GitTreeState:"clean"}
```

修复如下

```bash
➜  deployment git:(master) ✗ helm init --upgrade
$HELM_HOME has been configured at /Users/zhengjiajin/.helm.
Tiller (the Helm server-side component) has been upgraded to the current version.
➜  deployment git:(master) ✗ helm version
Client: &version.Version{SemVer:"v2.14.2", GitCommit:"a8b13cc5ab6a7dbef0a58f5061bcc7c0c61598e7", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.2", GitCommit:"a8b13cc5ab6a7dbef0a58f5061bcc7c0c61598e7", GitTreeState:"clean"}
➜  deployment git:(master) ✗ helm install stable/node-problem-detector --name node-problem-detector
```


## tiller 权限不足

```bash
helm install --name nginx stable/node-problem-detector
Error: release nginx failed: namespaces "default" is forbidden: User "system:serviceaccount:kube-system:default" cannot get namespaces in the namespace "default"
```

如上，tiller 使用 kube-system 分区下的 default serviceaccount 调用了 default 分区相关的 API 报了权限不足


卸载 tiller
```bash
helm reset
```

为 tiller 创建 rbac 规则，创建一个名称为 tiller 的 service account，并分配 cluster-admin 的 role

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

重新初始化 tiller
```bash
helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.7.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```
