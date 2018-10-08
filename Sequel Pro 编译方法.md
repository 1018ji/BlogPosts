---
title: Sequel Pro 编译方法
date: 2018-10-08 19:54:23
categories: Mac
tags:
  - Mac
  - Software
---

# Sequel Pro 简介

Sequel Pro 是轻量级 MySQL 管理软件，推荐使用

其他参考资料
1. Sequel Pro 官网 [点我](http://www.sequelpro.com/)
2. Sequel Pro 源码 [点我](https://github.com/sequelpro/sequelpro)
3. Sequel Pro Test Builds [点我](https://sequelpro.com/test-builds)

**官方 1.2 版本遥遥无期，官方 Test Builds 很久不更新，没办法自己去编译源码呗**

-----------------

<!-- more -->

# Sequel Pro 编译方法

步骤一 克隆代码

```
$ git clone https://github.com/sequelpro/sequelpro.git --depth=1
$ cd sequelpro
```
步骤二 修改编译配置文件 Debug 修改为 Release

```
$ sed -i '' -e 's/Debug/Release/g' Makefile
```
步骤三 移除 i386 支持（自 7d9ba0f87236cf4ed2eb0d6527311a872372c4c1 开始不再支持 i386）

```
$ find . -type f -name "*.pbxproj" -exec sed -i '' -e 's/ARCHS_STANDARD_32_64_BIT/ARCHS_STANDARD_64_BIT/g' {} +
```
步骤四 编译

```
$ make
```
步骤四 拷贝至 Application 文件夹

```
$ cp -R build/Release/Sequel\ Pro.app /Applications/Sequel\ Pro.app
```

-----------------
参考资料
1. khanhicetea 博客 [点我](https://khanhicetea.com/post/build-sequel-pro-from-source-in-xcode-10/)

-----------------

The END！