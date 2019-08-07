---
title: "搭建 tensorflow 开发环境"
date: 2019-02-25T16:34:28+08:00
tags: [ "tensorflows" ]
categories: [ "tensorflows" ]
---

tensorflow 官方没有教程教你如何去搭建 dev 环境，相比于 kubernetes 这方面文档不够全。这篇 blog 讲如何搭建一个开发者环境(想要深入了解代码，后续自己写一个 mini 版本的 tensorflow)，git clone 后的 tensorflow 包是无法直接被代码引用的，但是使用 pip install tensorflow 代码会在 pip package 里面。<!--more--> 在 pip 里面不方便看 tensorflow 源码，因为跳入到代码里面不知道这个文件在真实 repo 中的哪里目录中，同时修改没有办法看到效果。理想的方式是进入代码的时候能看到跳到 clone 后的源码库。另外可以方便不断的追踪 master 分支的代码。

> mac 需要安装 xcode

1.安装 bazel 并直接 bazel sync
```bash
brew tap bazelbuild/tap
brew tap-pin bazelbuild/tap
brew install bazelbuild/tap/bazel
# 执行 bazel sync
bazel sync
```

2.创建 virtual environment
```bash
virtualenv ~/.virtualenvs/tf-dev-env --no-site-packages
source /Users/admin/.virtualenvs/tf-dev-env/bin/activate
# 有的时候会因为在 shell 中设置了 alias，导致 python 指向的路径不对，最好先检查一下
which python
```

3.编译 tensorflow 代码(时间较长，在我的 mac 上用了 3 个多小时)
编译可以参考 tensorflow [官方文档](https://www.tensorflow.org/install/source)，因为不同时间 tensorflow 的依赖包不一样
```bash
pip install -U --user pip six numpy wheel mock
pip install -U --user keras_applications==1.0.5 --no-deps
pip install -U --user keras_preprocessing==1.0.3 --no-deps
./configure
# --verbose_failures 会把编译的错误打印出来
bazel build --copt=-march=native -c opt //tensorflow/tools/pip_package:build_pip_package --verbose_failures
```

4.建立软连接并 setup tensorflow
```bash
(tf-dev-env) ➜  ~ cd /Users/admin/python/tensorflow
(tf-dev-env) ➜  tensorflow git:(master) mkdir _python_build
(tf-dev-env) ➜  tensorflow git:(master) cd _python_build
(tf-dev-env) ➜  tensorflow git:(master) ln -s ../tensorflow/tools/pip_package/* .
(tf-dev-env) ➜  tensorflow git:(master) ln -s ../bazel-bin/tensorflow/tools/pip_package/build_pip_package.runfiles/org_tensorflow/* .
(tf-dev-env) ➜  tensorflow git:(master) python setup.py develop
...
Using /Users/admin/.virtualenvs/tf-dev-env/lib/python3.5/site-packages
Finished processing dependencies for tensorflow==1.12.0
```

5.查看 tensorflow path
```bash
(tf-dev-env) ➜  ~ python
Python 3.5.4 (v3.5.4:3f56838976, Aug  7 2017, 12:56:33)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
Limited tf.compat.v2.summary API due to missing TensorBoard installation
>>> tf.__path__
['/Users/admin/python/tensorflow/_python_build/tensorflow/python/keras/api/_v1', '/Users/admin/.virtualenvs/tf-dev-env/lib/python3.5/site-packages/tensorflow_estimator/python/estimator/api/_v1', '/Users/admin/python/tensorflow/_python_build/tensorflow', '/Users/admin/python/tensorflow/_python_build/tensorflow/_api/v1']
```
