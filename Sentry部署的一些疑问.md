---
title: Sentry 部署的一些疑问
date: 2018-01-13 19:40:23
categories: 运维
tags:
  - PHP
  - Sentry
---

# Sentry Docker 部署

推荐使用 Sentry 部署 Sentry

1. Docker 安装，请参照 [Docker 文档](https://hub.docker.com/_/sentry/)
2. Sentry 安装，请参照 [Sentry 文档](https://hub.docker.com/_/sentry/)

部署完成后存在如下容器
1. sentry-redis（Sentry Redis 依赖）
2. sentry-postgres（Sentry PostgreSQL 依赖）
3. my-sentry（Sentry Web 服务）
4. sentry-cron（Celery 定时任务）
5. sentry-worker-1（Celery Worker）

![容器列表](https://web-blog-1252155400.cos.ap-beijing.myqcloud.com/blog_photo/2018/01/sentry_main.png)

**Sentry 支持分布式，可启动多个 Worker **

-----------------

<!-- more -->

# Sentry cleanup 清理过期数据

阅读 CHANGES，Sentry 在 7.6.0 移除 `cleanup` 的 Task，但是命令并没有移除
The `cleanup` task has been removed (the command is still available)

为保证**磁盘容量**需要定期执行 `cleanup`，但是并不会真正清理 PostgreSQL 中的数据，而只是将其中需删除的数据标记为 `DEAD`，如果需要释放这些数据则需要执行 `vacuumdb` 命令。

具体步骤为：
1. 登录 my-sentry 机器，执行 `sentry cleanup --days 30` 命令
2. 登录 sentry-postgres 机器，执行 `vacuumdb -U sentry -d postgres -v -f --analyze` 命令
3. 执行 `df -h` 查看磁盘信息

-----------------

# Sentry SMTP 发送邮件

因使用腾讯企业邮箱发送提示邮件，但是测试 Sentry 本身配置文件并没有 SSL 相关配置项

具体修复方案为：
1. 执行 `pip install django-smtp-ssl` 命令，安装 django-smtp-ssl
2. 将邮件的 backend 替换为 `django_smtp_ssl.SSLEmailBackend`

django-smtp-ssl Github 地址：[django-smtp-ssl](https://github.com/bancek/django-smtp-ssl)

Sentry 配置文件参照如下

```
mail.backend: 'django_smtp_ssl.SSLEmailBackend'
mail.host: 'smtp.exmail.qq.com' # 腾讯企业邮 SMTP 服务器
mail.port: 465
mail.username: 'example@qudian.com'
mail.password: '我是密码'
mail.use-tls: false # 需设为 false 
mail.from: 'example@qudian.com' # mail.from 必须与 mail.username 一致，可能受服务商影响
```
-----------------

# Sentry 使用钉钉插件

Sentry 安装钉钉插件使 project 支持 钉钉机器人推送信息
**Sentry 默认会发送提示邮件，但对于统一错误不重复发送**

使用插件 sentry-dingtalk，Github 地址：[sentry-dingtalk](https://github.com/gzhappysky/sentry-dingtalk)

具体安装步骤为：
1. 执行 `pip install sentry-dingtalk` 命令，安装 sentry-dingtalk
2. 如第一步执行完成，则无需阅读后续操作。如第一步执行失败，推荐手动安装
3. 进入目录 `/home/sentry/data` (可自定义)，执行 `git clone https://github.com/gzhappysky/sentry-dingtalk.git`
4. 执行 `pip install /home/sentry/data/sentry-dingtalk` 手动安装 sentry-dingtalk
5. sentry-dingtalk 将被安装至 `/usr/local/lib/python2.7/site-packages/sentry_dingtal` 目录

界面配置：
1. 对于 Project 进行相关设置，`Project Settings（项目设置）` -> `Integration (所有集成)` -> 开 dingtalk 插件，点击保存点击
2. 在 `CONFIGURATION（集成`） 的 `Alerts（警报）` 或者 `Integration (所有集成)` 的 `dingtalk` 设置 `dingtalk robot url`
3. 进入 `CONFIGURATION（集成`） 的 `Alerts（警报）` 页面点击 `dingtalk` 栏目的 `Test Plugin` 测试页面

![测试图片](https://web-blog-1252155400.cos.ap-beijing.myqcloud.com/blog_photo/2018/01/sentry_dingtalk.png)






