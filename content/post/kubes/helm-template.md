---
title: "Helm Template 实践"
date: 2019-08-20T20:48:21+08:00
lastmod: 2019-08-20T20:48:21+08:00
tags: [ "kubernetes" ]
categories: [ "kubernetes" ]
---
## 前提
helm 使用 chart 管理 kubernetes yaml 文件，在做 kubernetes CI、CD 的时候需要部署和修改你的 yaml 文件。最开始的时候使用类似下面的方式修改 yaml 文件。

<!--more-->
```
kubectl set image -f deploy.yaml <container_name>=<image> --local -o yaml | kubectl apply -f -
```
因为比较经常改动的只有镜像名称，yaml 文件放在了用户的仓库中管理。后来遇到了很多的需求，比如支持日志，监控，ingress 的自动化配置，上面的就不太合适了，只能通过不断的打 patch 去管理你的 yaml 文件。后面考虑使用 sed 去打 patch，但是用的时候发现太丑了，就决定使用 chart 了。

使用 chart 比较担心的事情就是加大复杂度，helm有一个 server 端 tiller，历史版本通过 configmap 来管，这个也加大了维护的成本，和原生的管理方式已经不一样了。由于这几个原因最后决定只使用 helm 的 client 去渲染模版的功能，提供一个比较通用的 golang 应用模版，values.yaml 文件由用户来控制。不使用任何的 server 端的功能。

## helm template

使用 helm template  的好处是比较标准，相比于使用 patch 的方式，values 文件可以继续保留 yaml 结构。因为大部分应用的配置都长的差不多，只需要提取部分的字段来填写即可，可以大大的加速容器化构建和服务改造的成本。

helm template 看起来很复杂，但是实际在使用的时候其实挺简单的，看一两个例子就可以了。常用的语法如下：

### 引用 values

``````yaml
{{ .Values.name }}
``````

### If else 

```
{{- if .Values.environment }}
  namespace: "{{ .Values.team }}-{{ .Values.environment }}"
{{- else }}
  namespace: {{ .Values.team }}
{{- end }}
```
### to yaml
```
{{- if .Values.service.annotations }}
  annotations:
{{ toYaml .Values.service.annotations | indent 4 }}
{{- end }}
```
### for range
```
{{- range $pkey, $pval := .Values.env }}
        - name: {{ $pkey }}
          value: {{ quote $pval }}
{{- end }}
```

编写过程比较简单，学习起来几乎没有什么成本。最后通过 `helm template` 命令渲染就可以了。
```bash
helm template _chart/chart -x templates/namespace.yaml -f values.yaml --set environment=$ENV
```

下面的文件是一个基本的 golang 应用在自动化了监控、日志、ingress 下的情况下需要填写的 values.yaml 文件
```yaml
# required
name: "<服务名称>"

# required
# 目前通过 team-environment 作为分区名称，dev 和 qa 在一个集群，pre 和 prd 在一个集群
team: "<团队名称>"

# note: 如果有对外暴露的端口则必须添加健康检查，没有则不能添加健康检查，否则会导致应用无法运行
# 应用对外暴露的端口，如果没有注释掉
service:
  port: 11505
  targetPort: 11505
  nodePort: 32191

# 服务的健康检查，如果是一个需要对外暴露接口的服务，必须提供监控检查，可以避免滚动升级和扩缩容的时候出问题。
probePath: /ping
livenessProbe:
  initialDelaySeconds: 60
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1
readinessProbe:
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1

# optional
# TRACE，是否使用 tracing，tracing 目前是文件类型的日志，如果使用会去采集/data/log/ 下的 tracing 日志
trace: "true"

# optional
# 监控的端口和路径，会创建一个 service-metrics 的 svc，labels 的作用是定义被哪个 prome01 采集
monitoring:
  port: 11505
  path: /metrics
  labels:
    prometheus: "prome01"

# optional
# 自定义环境变量
env:
  APP_NAME: "<服务名称>"

# required
# 应用的副本数量，无状态应用可多副本
replicaCount: 1

# required
# 资源配额
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 1
    memory: 1Gi

# optional
# 水平扩容，下面表示当 cpu 使用率超过百分之 80 自动扩容，最小为 1，最大为 3
hpa:
  min: 1
  max: 3
  cpu_percent: 80

# 现在是一个二级的统配域名，部署后会自动通过 ingress 对外保留服务。pre 使用 ingress，prd 流量小的使用 ingress，流量大的直接使用 lb
ingress:
  wildcard_dns: <dns>

# 只有 environment=prd 才会创建 loadbalancer service
loadbalancer:
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-id: "<your-loadbalancer-id>"

environment: prd
```

最后在发布的时候需要执行的命令如下，从原来的几十行 patch 变成了 3 行 helm 命令。

```yaml
    stage('Deploy ...') {
      steps {
        container('go') {
          dir('_chart') {
            git branch: 'master', url: '<chart-git-dir>', credentialsId: 'gitlab-admin-ssh'
          }
          sh "cat values.yaml"
          sh "helm template _chart/chart -x templates/namespace.yaml -f values.yaml --set environment=$ENV| kubectl apply -f -"
          sh "helm template _chart/chart -f values.yaml --set environment=$ENV| kubectl apply -f -"
          sh "helm template _chart/chart -f values.yaml -x templates/deployment.yaml --set environment=$ENV| kubectl rollout status -f -"
        }
      }
    }
```

现在会把 chart 放在了另一个仓库中，然后在用户的仓库中管理 value.yaml 文件，通过 values.yaml 渲染模版然后再发布。
