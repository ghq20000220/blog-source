---
title: 利用github.io搭建blog
date: 2026-07-13 08:16:37
tags:
---

# 环境（windows版本）
1. git安装

    - 官方地址：<https://git-scm.cn/install/windows>
    - 安装后使用cmd或git bash输入`git --version`，输出版本，说明安装成功

2. nodejs安装
    - 官方地址：<https://nodejs.org/en/download/>
    - `node -v`,显示版本说明成功安装

3. Hexo安装
    - `npm install hexo -g`
    - `hexo -v`，测试安装状态

4. 安装Hexo依赖
    - `npm install --save hexo-deployer-git`

# git ssh配置
1. 生成ssh key
    - `ssh-keygen -t rsa -C "邮件地址"` ，连续回车。生成的内容在id_rsa.pub中
2. 将ssh key加入githu中
    - 使用`ssh -T git@github.com`测试ssh key的状态
3. 配置git user.name user.email
```
$ git config --global user.name "liyunchen" #你的github用户名
$ git config --global user.email "xxx@163.com" #填写你的github注册邮箱
```
# 初始化blog仓库
1. 初始化blog目录
    - hexo init
    - 可能会出现git clone失败问题
    - 可能会出现npm install安装卡住问题，可替换npm源
2. 生成静态网页: `hexo g`
3. 预览网页: `hexo s`
4. 发布至github: `hexo d`

# 编写blog
1. 新建blog
    - `hexo new 'blog-name'`

# bug合集
1. hexo 安装后，无法正常显示图片，需要安装插件
```
npm install https://github.com/CodeFalling/hexo-asset-image -- save
```



