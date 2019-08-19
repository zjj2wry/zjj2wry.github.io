---
title: "记录一些经常用到的 tips"
date: 2019-06-21T16:29:01+08:00
---

<!--more-->
## 修改默认 shell
```
# 查看支持的 shells
cat /etc/shells
# 修改默认的 shell
chsh -s /usr/local/bin/bash
```

## go mod 替换包
使用了第3方库，但是对第3方库做了改动，不希望修改现有代码里的 import
```bash
go mod edit -replace "package=package@version"
```

## 安装 kubectl bash complete
```bash
echo 'source <(kubectl completion zsh)' >>~/.zshrc
source ~/.zshrc
```

