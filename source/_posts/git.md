---
title: "git 常用命令"
date: 2016-06-23 00:00:00
cover: https://image.cdn.renzhaosy.cn/random-img/1.jpg
tags:
  - 笔记
---

# git 常用命令

### git 生成ssh key

```shell
ssh-keygen -t rsa -C "your_email@example.com"
```

### git 用户信息

```shell
git config --global user.name "Joiner"
git config --global user.email joiner@gmail.com
```

### git 更新 .gitignore 文件

```shell
    git rm -r --cached .  #清除缓存 
    git add . #重新trace file  
    git commit -m "update .gitignore" #提交和注释
    git push origin master #可选，如果需要同步到remote上的话
```

### git 创建本地分支与远程分支关联
用 git branch --set-upstream 命令

```shell
git branch --set-upstream-to=origin/<branch> master 

git push -u origin <branch>
```

### git 丢弃本地修改

```shell
#丢弃本地修改
git checkout -- file 

#丢弃本地全部修改
git check -- .

#移除本地添加文件
git clean -df //-d表示同时移除目录,-f表示force,因为在git的配置文件中, clean.requireForce=true,如果不加-f,clean将会拒绝执行.

git clean 参数
    -n 显示 将要 删除的 文件 和  目录
    -f 删除 文件
    -df 删除 文件 和 目录

```

### 常用命令


```shell

# 列出所有本地分支
$ git branch

# 列出所有远程分支
$ git branch -r

# 列出所有本地分支和远程分支
$ git branch -a

# 新建一个分支，但依然停留在当前分支
$ git branch [branch-name]

# 新建一个分支，并切换到该分支
$ git checkout -b [branch]

# 新建一个分支，指向指定commit
$ git branch [branch] [commit]

# 新建一个分支，与指定的远程分支建立追踪关系
$ git branch --track [branch] [remote-branch]

# 切换到指定分支，并更新工作区
$ git checkout [branch-name]

# 切换到上一个分支
$ git checkout -

# 建立追踪关系，在现有分支与指定的远程分支之间
$ git branch --set-upstream [branch] [remote-branch]

# 合并指定分支到当前分支
$ git merge [branch]

# 选择一个commit，合并进当前分支
$ git cherry-pick [commit]

# 删除分支
$ git branch -d [branch-name]

# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]
```

