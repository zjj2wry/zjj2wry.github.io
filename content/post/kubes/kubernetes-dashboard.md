---
title: "部署 Kubernetes Dashboard"
date: 2019-10-17T19:43:58+08:00
lastmod: 2019-10-17T19:43:58+08:00
---

搭建 dashboard 可以通过官方仓库中 README 提供的命令快速部署`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml`，也可以通过 helm 安装。<!--more-->在使用官方库安装的时候会默认设置 `--auto-generate-certificates=true`开启 https 端口。只能通过 kubectl proxy 来访问。通过 helm 安装比较灵活，同时可以设置 ingress+https。

[helm chart 的仓库地址](https://github.com/helm/charts/tree/master/stable/kubernetes-dashboard)，自定义 values.yaml 文件。

```yaml
# Default values for kubernetes-dashboard
# This is a YAML-formatted file.
# Declare name/value pairs to be passed into your templates.
# name: value

image:
  repository: k8s.gcr.io/kubernetes-dashboard-amd64
  tag: v1.10.1
  pullPolicy: IfNotPresent
  pullSecrets: []

replicaCount: 1

## Here annotations can be added to the kubernetes dashboard deployment
annotations: {}
## Here labels can be added to the kubernetes dashboard deployment
##
labels: {}
# kubernetes.io/name: "Kubernetes Dashboard"


## Enable possibility to skip login
enableSkipLogin: false

## Serve application over HTTP without TLS
enableInsecureLogin: true

## Additional container arguments
##
# extraArgs:
#   - --enable-skip-login
#   - --enable-insecure-login
#   - --system-banner="Welcome to Kubernetes"

## Additional container environment variables
##
extraEnv: []
# - name: SOME_VAR
#   value: 'some value'

# Annotations to be added to kubernetes dashboard pods
## Recommended value
# podAnnotations:
#   seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
podAnnotations: {}

## SecurityContext for the kubernetes dashboard container
## Recommended values
# dashboardContainerSecurityContext:
#   allowPrivilegeEscalation: false
#   readOnlyRootFilesystem: true
## The two values below can be set here or at podLevel (using variable .securityContext)
#   runAsUser: 1001
#   runAsGroup: 2001
dashboardContainerSecurityContext: {}

## Node labels for pod assignment
## Ref: https://kubernetes.io/docs/user-guide/node-selection/
##
nodeSelector: {}

## List of node taints to tolerate (requires Kubernetes >= 1.6)
tolerations: []
#  - key: "key"
#    operator: "Equal|Exists"
#    value: "value"
#    effect: "NoSchedule|PreferNoSchedule|NoExecute"

## Affinity
## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
affinity: {}

# priorityClassName: ""

service:
  type: ClusterIP
  externalPort: 9090

  ## This allows an override of the heapster service name
  ## Default: {{ .Chart.Name }}
  ##
  # nameOverride:

  # LoadBalancerSourcesRange is a list of allowed CIDR values, which are combined with ServicePort to
  # set allowed inbound rules on the security group assigned to the master load balancer
  # loadBalancerSourceRanges: []

  ## Kubernetes Dashboard Service annotations
  ##
  ## For GCE ingress, the following annotation is required:
  ## service.alpha.kubernetes.io/app-protocols: '{"https":"HTTPS"}' if enableInsecureLogin=false
  ## or
  ## service.alpha.kubernetes.io/app-protocols: '{"http":"HTTP"}' if enableInsecureLogin=true
  annotations: {}

  ## Here labels can be added to the Kubernetes Dashboard service
  ##
  labels: {}
  # kubernetes.io/name: "Kubernetes Dashboard"

resources:
  limits:
    cpu: 100m
    memory: 100Mi
  requests:
    cpu: 100m
    memory: 100Mi

ingress:
  ## If true, Kubernetes Dashboard Ingress will be created.
  ##
  enabled: true

  ## Kubernetes Dashboard Ingress annotations
  ##
  ## Add custom labels
  # labels:
  #   key: value
  # annotations:
  #   kubernetes.io/ingress.class: nginx
  #   kubernetes.io/tls-acme: 'true'
  ## If you plan to use TLS backend with enableInsecureLogin set to false
  ## (default), you need to uncomment the below.
  ## If you use ingress-nginx < 0.21.0
  #   nginx.ingress.kubernetes.io/secure-backends: "true"
  ## if you use ingress-nginx >= 0.21.0
  #   nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"


  ## Kubernetes Dashboard Ingress paths
  ##
  paths:
    - /
  #  - /*

  ## Kubernetes Dashboard Ingress hostnames
  ## Must be provided if Ingress is enabled
  ##
  hosts:
    - kubernetes-dashboard.domain.com

  ## Kubernetes Dashboard Ingress TLS configuration
  ## Secrets must be manually created in the namespace
  ##
  # tls:
  #   - secretName: kubernetes-dashboard-tls
  #     hosts:
  #       - kubernetes-dashboard.domain.com

rbac:
  # Specifies whether RBAC resources should be created
  create: true

  # Specifies whether cluster-admin ClusterRole will be used for dashboard
  # ServiceAccount (NOT RECOMMENDED).
  clusterAdminRole: false

  # Start in ReadOnly mode.
  # Only dashboard-related Secrets and ConfigMaps will still be available for writing.
  #
  # Turn OFF clusterAdminRole to use clusterReadOnlyRole.
  #
  # The basic idea of the clusterReadOnlyRole comparing to the clusterAdminRole
  # is not to hide all the secrets and sensitive data but more
  # to avoid accidental changes in the cluster outside the standard CI/CD.
  #
  # Same as for clusterAdminRole, it is NOT RECOMMENDED to use this version in production.
  # Instead you should review the role and remove all potentially sensitive parts such as
  # access to persistentvolumes, pods/log etc.
  clusterReadOnlyRole: false

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

livenessProbe:
  # Number of seconds to wait before sending first probe
  initialDelaySeconds: 30
  # Number of seconds to wait for probe response
  timeoutSeconds: 30

podDisruptionBudget:
  # https://kubernetes.io/docs/tasks/run-application/configure-pdb/
  enabled: false
  minAvailable:
  maxUnavailable:


## PodSecurityContext for pod level securityContext
##
# securityContext:
#   runAsUser: 1001
#   runAsGroup: 2001
securityContext: {}

networkPolicy: false
```

可以通过在 ingress 配置 https, 所以设置 `enableInsecureLogin: true` 和 `externalPort: 9090`，如果有 https 证书可以修改 ingress.tls 的配置。

执行 helm install：

```bash
helm install stable/kubernetes-dashboard -f values.yaml --name=dashboard --namespace=dashboard
```

> 如果在运行的时候 dashboard 的日志报没有权限，设置 rbac.clusterAdminRole 为 true。
