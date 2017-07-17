title: Use nvm to manage node
tags:
  - Nodejs
categories:
  - Study
cc: true
hljs: true
comments: true
date: 2017-07-17 15:10:51
---
> 使用NVM来管理node的版本，并使用国内镜像安装

![Node](/images/nodejs.jpg)
## 安装[NVM](https://github.com/creationix/nvm)
1. 从GITHUB获取最新链接
* Use curl
```bash
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
```
* or use wget
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
```

2. 设置环境变量使用国内node镜像
```bash
$ vim ~/.bashrc

#set nodejs env
export NVM_NODEJS_ORG_MIRROR=http://npm.taobao.org/mirrors/node
```
3. 设置生效
```bash
$ source ~/.bashrc
```
4. 设置npm国内镜像
```bash
$ npm config set registry https://registry.npm.taobao.org
```
