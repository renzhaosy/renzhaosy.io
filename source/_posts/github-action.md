---
title: "GitHub Action部署Hexo静态页面"
cover: /img/github-action.jpg
tags:
  - 笔记
---
# GitHub Action部署Hexo静态页面

每次部署博客都要本地手动执行 `hexo deploy`，很麻烦，现在GitHub推出了GitHub Action可以自动化执行部署操作。

网上有很多创建GitHub Action的教程，这里就不一步一步说了，大家可以学习下阮一峰的 [GitHub Actions 入门教程](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)

部署脚本独立成一个action供大家使用 [renzhaosy/hexo-deploy-action](https://github.com/marketplace/actions/hexo-deploy-action)

workflows 配置如下

```yml

name: Deploy to Github.io
on:
  push:
    branches:
      - master
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Build and Deploy
      uses: renzhaosy/hexo-deploy-action@master
      env:
        PERSONAL_TOKEN: ${{ secrets.ACCESS_TOKEN }} # github 私人权限token
        PUBLISH_REPOSITORY: renzhaosy/renzhaosy.github.io # 打包之后想要部署到的仓库
        BRANCH: master # 打包之后想要部署到的分支
        PUBLISH_DIR: ./public  # hexo 打包之后文件的所在的文件夹
```

ACCESS_TOKEN 是 [github 私人权限token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line),生成之后添加到项目`Settings/Secrets`中就可以使用了。
