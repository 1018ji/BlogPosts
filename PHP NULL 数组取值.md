---
title: PHP NULL 数组取值
date: 2018-03-14 11:35:33
categories: Code
tags:
  - PHP
---
# PHP NULL 数组取值

废话少说，直接测试两种情况

测试网站为 [PHP HHVM 多版本测试](https://3v4l.org/)

-----------------

## 变量为 NULL，对变量进行数组取值操作

测试代码如下

```
<?php

echo "------------测试 null 变量进行数组取值---------\n";
$testArray = null;
var_dump($testArray[0]);
var_dump($testArray['abc']);
echo '------------END---------';
```
测试结果

```
Output for 4.3.0 - 5.6.30, hhvm-3.10.1 - 3.22.0, 7.0.0 - 7.2.3
------------测试 null 变量进行数组取值---------
NULL
NULL
------------END---------
```
各版本均返回 NULL

-----------------

## 对 NULL 进行数组取值操作

测试代码如下

```
<?php

echo "------------测试 null 直接进行数组取值---------\n";
var_dump(null[0]);
var_dump(null['abc']);
echo '------------END---------';
```
测试结果

```
Output for 5.6.0 - 5.6.30, hhvm-3.15.4 - 3.22.0, 7.0.0 - 7.2.3
------------测试 null 直接进行数组取值---------
NULL
NULL
------------END---------
```
```
Output for hhvm-3.11.1 - 3.13.2
Fatal error: Uncaught Error: syntax error, unexpected '[', expecting ')' in /in/0G3g2:4
Stack trace:
#0 {main}

Process exited with code 255.
```
```
Output for hhvm-3.10.1
Fatal error: syntax error, unexpected '[', expecting ')' in /in/0G3g2 on line 4

Process exited with code 255.
```
```
Output for 4.4.2 - 4.4.9, 5.1.0 - 5.5.38
Parse error: syntax error, unexpected '[' in /in/0G3g2 on line 4

Process exited with code 255.
```
```
Output for 4.3.0 - 4.3.1, 4.3.5 - 4.4.1, 5.0.0 - 5.0.5
Parse error: parse error, unexpected '[' in /in/0G3g2 on line 4

Process exited with code 255.
```
```
Output for 4.3.2 - 4.3.4
Parse error: parse error in /in/0G3g2 on line 4

Process exited with code 255.
```
PHP 版本 5.6 及以上不报错，HHVM 个版本报错

-----------------

## 结尾

PHP 各版本之间有些许不同，在实际使用还是需要注意的。

另外，使用 PHP 的语法规避代码中的判断检查，对于代码简洁还是很有必要的。

-----------------

**End**




