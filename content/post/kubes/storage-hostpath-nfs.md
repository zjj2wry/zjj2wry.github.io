---
title: "kuberntes host-path、nfs provisioner"
date: 2019-11-25T16:36:25+08:00
lastmod: 2019-11-25T16:36:25+08:00

---
场景是使用本地的目录作为持久卷或者想要将本地目录作为共享数据盘对外提供<!--more-->

## local path provisioner

local path provisioner 可以让你使用 pv pvc 的方式来使用本地目录，同时添加了 pv schedule，保证 pod 在升级的时候还会调度到原来的节点上。
部署 local path provisioner
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```
查看创建的 local-path storageclass，volumeBindingMode 是 WaitForFirstConsumer，WaitForFirstConsumer 和 Immediate 的区别是，Immediate 会在 pvc
创建后马上创建 pv，而 WaitForFirstConsumer 会在 pod 使用的时候才创建 pv，主要作用是让调度度能感知 pv 的情况来调度 pod。
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

设置为默认的 storageclass，如果系统中存在其他默认的 storageclass，需要把它们设置为 false
```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## nfs provisioner

部署 local path provisioner
> quota 好像没有作用, 最终的 quota 限制是目录所在的磁盘

```bash
helm install stable/nfs-server-provisioner --set persistence.enabled=true,persistence.size=30Gi --namespace=nfs
```

nfs provisioner 会使用一个 host path 的 volume 来作为共享盘来使用，查看创建出来的 pv 可以看到 nodeAffinity 信息，pod 在调度的
时候会感知到 pv 的调度限制，保证可以调度原来的 node 上。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-3d80384a-0f55-11ea-96b6-00163e129c28
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 30Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: data-ideal-nightingale-nfs-server-provisioner-0
    namespace: nfs
  hostPath:
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - cn-beijing.172.16.133.205
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
```

指定 storage class 为 nfs 创建共享盘即可
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
  name: nfs-pvc
  namespace: jx
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
  storageClassName: nfs
```

到节点查看文件信息
```bash
$ cd local-path-provisioner/
# 查看 local path 的 volume
$ ls
pvc-36c399a7-0f3f-11ea-9180-00163e059bee
$ cd pvc-36c399a7-0f3f-11ea-9180-00163e059bee/
$ ls
ganesha.log  nfs-provisioner.identity  pvc-911a998e-0f49-11ea-87dc-00163e0577f1  vfs.conf
# 查看 nfs 的 volume
$ ls pvc-911a998e-0f49-11ea-87dc-00163e0577f1/
agent  composer  support  workspace
$
```