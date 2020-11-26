---
title: Medis Mac Redis 客户端编译教程
date: 2020-11-26 16:09:37
categories: Medis Redis Mac
tags:
  - Mac
  - Redis
  - Software
---

# Medis 简介

Medis 一款运行在 Mac 上的 Redis 可视化客户端。

Medis 已上架 Apple Store，如非必要请支持正版应用，下载页面 [点我](https://apps.apple.com/app/medis-gui-for-redis/id1063631769)。


# 官网

* Medis 下载页面 [点我](https://github.com/luin/medis)


# 准备工作

安装 Node 程序 (nvm 方式)，根据最新源码，Node 版本不能超过 12，此次选用 10 版本

```
nvm install 10.15.3

nvm use 10.15.3

```

下载 Medis 源码

```
cd ~/Desktop

git clone https://github.com/luin/medis.git --depth=1

cd medis

```


# 修改源码

阻止其加载 webpack-bundle-analyzer 插件，否则会卡主无法继续打包

```
https://raw.githubusercontent.com/luin/medis/master/webpack.config.js

注释掉第 84 行 renderPlugins.push(new BundleAnalyzerPlugin())

```


# 编译

npm 编译

```
npm install

npm run pack
```

```
如果出现以下错误可忽略，没有签名导致的

Unhandled rejection Error: No identity found for signing.

```

编译后的文件位于目录 medis/dist/out/Medis-mas-x64/Medis，可以放在应用程序或者直接打开使用

注意自行编译的可能会覆盖非正常途径获取的 Medis 的配置，请慎重


The END！