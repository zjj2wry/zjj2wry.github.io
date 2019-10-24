---
title: "docker aufs 镜像驱动"
date: 2019-10-24T14:10:54+08:00
lastmod: 2019-10-24T14:10:54+08:00
keywords: []
description: ""
tags: []
categories: ["docker"]
author: ""
---

docker 镜像使用 linux 联合文件系统实现，联合文件系统的作用就是可以把多个目录挂载到同一个目录，在一个目录下管理所有的文件。docker 对镜像分层管理，分成只读层和可读写层。<!--more-->
只读层就是镜像层，读写层是容器层。只读层会被多个容器共享，为了保证不会被修改，会使用 copy-on-write 技术，当第一次对镜像层的文件修改时，会直接 copy 一份然后修改，保证底
层的基础镜像不被修改。这样的好处是可以节约容器运行时的空间和加快启动的速度。docker 默认推荐的镜像驱动是 overlay2, 第一代镜像驱动是 aufs。

# aufs

## aufs in centos
```bash
# 查看支持的文件驱动
cat /proc/filesystems
```

[centos 安装 aufs](https://www.jianshu.com/p/63fdb0c0659c), 将文中的`cd /etc/yum.repo.d`改为 `cd /etc/yum.repos.d` 测试可以安装成功。

```bash
mkdir aufs
cd aufs/
mkdir l1 l2 l3
echo "l1" > l1/container.txt
echo "l2" > l2/image2.txt
echo "l3" > l3/image3.txt
mkdir merged
sudo mount -t aufs -o dirs=./l1:./l2:./l3/ none ./merged/
```

执行 mount 的时候发生了点错误，google 一下也没有原因。官方文档 [Supported backing filesystems](https://docs.docker.com/storage/storagedriver/select-storage-driver/#supported-backing-filesystems) 显示 aufs 是支持 xfs 文件系统的，原因待查。
```bash
[root@localhost aufs]# sudo mount -t aufs -o dirs=./l1:./l2:./l3/ none ./merged/
mount: 文件系统类型错误、选项错误、none 上有坏超级块、
       缺少代码页或助手程序，或其他错误

       有些情况下在 syslog 中可以找到一些有用信息- 请尝试
       dmesg | tail  这样的命令看看。
[root@localhost aufs]# dmesg | tail
[   15.772053] Not activating Mandatory Access Control as /sbin/tomoyo-init does not exist.
[   66.816987] cfg80211: failed to load regulatory.db
[ 3704.286263] aufs au_xino_create:220:mount[12023]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 3728.939854] aufs au_xino_create:220:mount[12025]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 3821.676973] aufs au_xino_create:220:mount[12029]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 4071.906530] aufs au_xino_create:220:mount[12032]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 4077.942929] aufs au_xino_create:220:mount[12034]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 4083.080624] aufs au_xino_create:220:mount[12036]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 4087.756772] aufs au_xino_create:220:mount[12038]: xino doesn't support /tmp/.aufs.xino(xfs)
[ 5048.856785] aufs au_xino_create:220:mount[12076]: xino doesn't support /tmp/.aufs.xino(xfs)
```

mount 查看挂载信息，rootfs 使用了 xfs，
```bash
[root@localhost ~]# mount
/dev/mapper/centos-root on / type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

## aufs in ubuntu

安装 aufs
```bash
sudo apt-get install aufs-tools
```

测试 aufs mount
```bash
mkdir aufs
cd aufs/
mkdir l1 l2 l3
echo "l1" > l1/container.txt
echo "l2" > l2/image2.txt
echo "l3" > l3/image3.txt
mkdir merged
sudo mount -t aufs -o dirs=./l1:./l2:./l3/ none ./merged/
```

查看 aufs 挂载点信息
```bash
vagrant@dev:~/aufs$ cat /sys/fs/aufs/si_88135210fab8cc26/*
/home/vagrant/aufs/l1=rw
/home/vagrant/aufs/l2=ro
/home/vagrant/aufs/l3=ro
64
65
66
/home/vagrant/aufs/l1/.aufs.xino
```

查看 merged 目录信息，l1 l2 l3 都挂载到 merged 目录
```bash
vagrant@dev:~/aufs$ ls merged/
container.txt  image2.txt  image3.txt
```

修改 merged 目录下的读写层, l1/container.txt 内容也发生了修改
```bash
vagrant@dev:~/aufs$ echo "update" >> ./merged/container.txt
vagrant@dev:~/aufs$ cat merged/container.txt 
l1
update
vagrant@dev:~/aufs$ cat l1/container.txt 
l1
update
```
修改 merged 目录下只读层, 只读层的内容没有发生修改, 读写层 copy 了一份并修改了里面的内容
```bash
vagrant@dev:~/aufs$ echo "update" >> ./merged/image2.txt
vagrant@dev:~/aufs$ ls merged/
container.txt  image2.txt  image3.txt
vagrant@dev:~/aufs$ cat merged/image2.txt 
l2
update
# 只读层 l2 内容没有发生修改
vagrant@dev:~/aufs$ cat l2/image2.txt 
l2
# 读写层 l1 中多了一个文件 image2.txt
vagrant@dev:~/aufs$ ls l1
container.txt  image2.txt
vagrant@dev:~/aufs$ cat l1/image2.txt 
l2
update
```

删除 merged 目录下只读层
```bash
vagrant@dev:~/aufs$ rm merged/image3.txt
vagrant@dev:~/aufs$ ls merged/
container.txt  image2.txt
# 多了一个 .wh.image3.txt 的文件
vagrant@dev:~/aufs$ ls l1/
container.txt   image2.txt      .wh.image3.txt  .wh..wh.aufs    .wh..wh.orph/   .wh..wh.plnk/
l3 的内容没有任何的修改
vagrant@dev:~/aufs$ ls l3/image3.txt 
l3/image3.txt
```


# ref

[storage drivers](https://docs.docker.com/storage/storagedriver/)
