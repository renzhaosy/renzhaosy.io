---
title: "Nodenv管理Node版本"
date: 2019-04-5 21:00:00
cover: https://image.cdn.renzhaosy.cn/post/node.jpeg
tags:
  - 工具
  - node
---

# Nodenv管理Node版本

## node安装 nodenv

使用 [nodenv](https://github.com/nodenv/nodenv) 安装node

### 1. 安装nodenv

```shell
$ brew install nodenv
```

### 2. 配置

#### 2.1 在 .bash_profile 中添加配置

```shell
export PATH="/Users/joiner/.nodenv/shims:$PATH"

eval "$(nodenv init -)"
```
#### 2.2 Installs autocompletion

```shell
$ source ~/.nodenv/completions/nodenv.zsh

# 如果没有 ~/.nodenv/completions/nodenv.zsh 文件手动创建 或者 从安装目录拷贝 (`/usr/local/Cellar/nodenv/1.3.0/completions`)

```

nodenv.zsh 内容如下

```vim

if [[ ! -o interactive ]]; then
    return
fi

compctl -K _nodenv nodenv

_nodenv() {
  local words completions
  read -cA words

  if [ "${#words}" -eq 2 ]; then
    completions="$(nodenv commands)"
  else
    completions="$(nodenv completions ${words[2,-2]})"
  fi

  reply=("${(ps:\n:)completions}")
}

```

#### 2.3 手动执行 nodenv init -

### 3. 测试

```shell
$ curl -fsSL https://github.com/nodenv/nodenv-installer/raw/master/bin/nodenv-doctor | bash
Checking for `nodenv' in PATH: /usr/local/bin/nodenv
Checking for nodenv shims in PATH: OK
Checking `nodenv install' support: /usr/local/bin/nodenv-install (node-build 4.6.1)
Counting installed Node versions: 2 versions
Auditing installed plugins: OK
```
## 使用

#### install node versions

```shell
# list all available versions:
$ nodenv install -l
# install a Node version:
$ nodenv install 9.9.0
# uninstall a node version:
$ nodenv uninstall 9.9.0
```

#### 设置 node 版本

```shell
# 设置全局node 版本
$ nodenv install 10.0.0
# 设置当前文件夹node 版本
$ nodenv local 9.9.0
# 取消当前文件夹node 版本设置
$ nodenv local --unset
```

#### nodenv versions

```shell
 9.9.0
* 10.16.0 (set by /Users/joiner/.nodenv/version)
```

#### 更新 nodenv node-build

```shell
brew upgrade nodenv node-build

```

#### 获取新版本

```shell
# 查看可用node版本列表
$ node-build --definitions
# or
$ nodenv install -l
```

node-build 有时候node的版本列表没有最新的版本，可以安装 `node-build-update-defs` 插件，将`node-build` 中的版本列表替换掉。

```shell
$ brew install nodenv/nodenv/node-build-update-defs
# 设置变量
$ export NODE_BUILD_DEFINITIONS=$(brew --prefix node-build-update-defs)/share/node-build
# 更换
$ nodenv update-version-defs
```

### 6. 更换 nodejs 源

下载nodejs的时候可能会很慢，可以用阿里的源

```shell
$ export NODE_BUILD_MIRROR_URL=https://npm.taobao.org/mirrors/node
```

