---
title: "使用 kubernetes 运行 tensorflow 分布式训练时的常见问题"
date: 2019-02-11T14:33:25+08:00
tags: [ "tensorflows","tips","distributed" ]
categories: [ "tensorflows","kubernetes" ]
---

### worker 执行完任务后没有正常退出(seession close 失败)
tensorflow 分布式训练可以使用 Supervisor 和 MonitoredTrainingSession, 后者是 tensorflow 推荐的方式，使用 tf-operator 运行分布式训练的时候，训练结束后 worker 没有正常退出，因为都是容器，导致的结果是资源无法正常回收。<!--more--> 

详情见 [issue](https://github.com/tensorflow/tensorflow/issues/21745)

#### 解决方法

分布式训练的时候只需要 worker 和 ps 通信，详情见 [issue](https://github.com/linkedin/TonY/pull/120/files)

```python
config_proto = tf.ConfigProto(device_filters = ['/job:ps', '/job:worker/task:%d' % task_index])
...
with tf.train.MonitoredTrainingSession(master=server.target,
                                       is_chief=(task_index == 0),
                                       checkpoint_dir=FLAGS.working_dir,
                                       hooks=hooks,
                                       config=config_proto) as sess:
```

### 训练的时候没有输出日志
python 的 stdout 是带缓冲的，在跑 k8s 分布式训练的时候经常出现本地运行的时候有日志，但是使用容器运行的时候没有日志。

#### 解决方法

1. 执行 python 程序的时候加一个 -u, 比如 ```python -u main.py```
2. 通过传递环境变量 ```PYTHONUNBUFFERED=1```
