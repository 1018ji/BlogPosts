---
title: Linux Core dump
date: 2018-02-27 15:00:33
categories: 运维
tags:
  - Linux
  - Core dump
---
# Linux Core Dump

当程序运行的过程中异常终止或崩溃，操作系统会将程序当时的内存状态记录下来，保存在一个文件中，这种行为就叫做Core Dump。Core Dump 除了记录内存信息之外，关键的程序运行状态也会同时 dump 下来，例如寄存器信息（包括程序指针、栈指针等）、内存管理信息、其他处理器和操作系统状态和信息。

Core Dump 对于编程人员诊断和调试程序是非常有帮助的，因为对于有些程序错误是很难重现的，例如指针异常，而 Core Dump 文件可以再现程序出错时的情景。

**本文基于 Debian 9.3 进行测试，也会介绍 Mac 一些不同**

-----------------

# Linux Core 打开关闭

列出所有资源的限制 命令

```
ulimit -a
```
列出所有资源的限制 输出

```
# ulimit -a
time(seconds)        unlimited
file(blocks)         unlimited
data(kbytes)         unlimited
stack(kbytes)        8192
coredump(blocks)     unlimited
memory(kbytes)       unlimited
locked memory(kbytes) 82000
process              unlimited
nofiles              1048576
vmemory(kbytes)      unlimited
locks                unlimited
rtprio               0
```
或者使用

查看 core file size 命令

```
ulimit -c
```
列出所有资源的限制 输出

```
# ulimit -c
0
```
core file size 解释：
* `0` 程序出错时不会产生 core 文件
* `1024` 程序出错时产生 core 文件不能超过 1024K
* `unlimited` 程序出错时会产生 core 文件大小不受限制

默认为 `0`，可将其设置为 `unlimited`，待解决后再将其置为 `0`，设置命令：

```
ulimit -c unlimited
```

**尽量将 core file size 设置得大一些，程序崩溃时生成 Core 文件大小即为程序运行时占用的内存大小。在发生堆栈溢出的时候，占用更大的内存**

-----------------

# Linux Core 文件名称以及路径设置

默认生成路径：输入可执行文件运行命令的同一路径下
默认生成名字：默认命名为 core 。新的 core 文件会覆盖旧的core文件

## 设置 PID 作为文件扩展名

查看命令

```
# cat /proc/sys/kernel/core_uses_pid
0
```
设置命令

```
echo 1 > /proc/sys/kernel/core_uses_pid
```
参数解释

* `0` 不添加 pid 作为扩展名，生成的 core 文件名称为 core
* `1` 添加 pid 作为扩展名，生成的 core 文件名称为 core.pid

## 设置文件名称以及保存位置

查看命令

```
# cat /proc/sys/kernel/core_pattern
core
```
设置命令

```
echo "/tmp/core-%e-%p-%t" > /proc/sys/kernel/core_pattern
```
参数解释
* `%p` - insert pid into filename 添加pid(进程id)
* `%u` - insert current uid into filename 添加当前uid(用户id)
* `%g` - insert current gid into filename 添加当前gid(用户组id)
* `%s` - insert signal that caused the coredump into the filename 添加导致产生core的信号
* `%t` - insert UNIX time that the coredump occurred into filename 添加core文件生成时的unix时间
* `%h` - insert hostname where the coredump happened into filename 添加主机名
* `%e` - insert coredumping executable name into filename 添加导致产生core的命令名

**本文使用 docker debian 如果无法修改系统参数，请添加 --privileged 参数**

<!-- more -->

-----------------

# Linux Core 文件测试

使用 kill 命令生成 Core 文件

```
kill -s SIGSEGV PID
```
**将会在 /proc/sys/kernel/core_pattern 配置的路径找到 Core 文件**

-----------------

# Linux Core 调试工具 GDB

要分析 Core 文件，需要使用 GDB

建议分析命令

```
gdb -e php-fpm -c core.pid
```
**php-fpm 建议使用绝对路径，core.pid 为生成的 core 文件**

进入 gdb 后，使用 bt 命令（显示程序堆栈信息），查找异常

gdb 教程请参照：[教程](http://blog.csdn.net/gatieme/article/details/51671430)

-----------------

# Mac 的一些疑问：

## 1. GDB 签名异常

gdb 使用 attach pid 调试正在运行进程时，报如下错误

```
Unable to find Mach task port  for  process-id 28885: (os/kern) failure (0x5). 
(please check gdb is codesigned - see taskgated(8))
```
异常中说明 gdb 需要进行签名，参照签名教程：[教程实例一](https://jingyan.baidu.com/article/15622f241db565fdfcbea515.html)、[教程实例二](https://segmentfault.com/a/1190000004136351)

## 2. Mac 生成证书提示位置错误

依据上述教程生成证书时可能会出现证书错误，建议在最后一步`钥匙串`选择`登录`而不是`系统`

**生成证书后，点击`钥匙串`的`登录`文件夹，将生成的证书复制，然后点击`系统`文件夹，执行粘贴操作，就可以绕过此问题**

## 3. Mac 系统 Core 文件保存位置

Mac 系统并没有上述的 `/proc/sys/kernel/core_uses_pid` 和 `/proc/sys/kernel/core_pattern` 文件

使用如下命令很容易获取 Core 文件保存位置以及各式

```
# sysctl -a | grep core
kern.corefile: /cores/core.%P
kern.coredump: 1
kern.sugid_coredump: 0
machdep.cpu.cores_per_package: 8
machdep.cpu.thermal.core_power_limits: 1
machdep.cpu.core_count: 2
```
Core 文件存放位置为 /cores
Core 文件各式为 core.%P

## 4. Mac Core Dump 官方教程

官方教程：[地址](https://developer.apple.com/library/content/technotes/tn2124/_index.html#//apple_ref/doc/uid/DTS10003391-CH1-SECCOREDUMPS)







