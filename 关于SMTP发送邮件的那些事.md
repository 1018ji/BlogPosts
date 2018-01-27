---
title: 关于 SMTP 发送邮件的那些事
date: 2018-01-24 13:16:13
tags:
---

关于 SMTP 发送邮件的那些事
===

![邮件](http://dec3.jlu.edu.cn/webcourse/T000277/files/images/070.jpg)

-------------

## SMTP简介
SMTP称为简单邮件传输协议（Simple Mail Transfer Protocal），目标是向用户提供高效、可靠的邮件传输。它的一个重要特点是它能够在传送中接力传送邮件，即邮件可以通过不同网络上的主机接力式传送。通常它工作在两种情况下：一是邮件从客户机传输到服务器；二是从某一个服务器传输到另一个服务器。SMTP是一个请求/响应协议，它监听25号端口，用于接收用户的Mail请求，并与远端Mail服务器建立SMTP连接。

## SMTP连接发送过程
1. 建立TCP连接
2. 客户端发送HELO命令以标识发件人自己的身份
3. 客户端发送MAIL命令
4. 客户端发送RCPT命令，以标识该电子邮件的接收人，可以存在多个RCPT行
5. 协商结束，准备发送邮件，用命令DATA发送
6. 以.表示结束，邮件发送
7. 使用QUIT命令退出

## ESMTP(Extended SMTP)连接发送过程
1. 建立TCP连接
2. 客户端发送HELO命令以标识发件人自己的身份
3. 客户端发送AUTH LOGIN命令，进行身份验证
4. 客户端发送MAIL命令
5. 客户端发送RCPT命令，以标识该电子邮件的接收人，可以存在多个RCPT行
6. 协商结束，准备发送邮件，用命令DATA发送
7. 以.表示结束，邮件发送
8. 使用QUIT命令退出

## 使用Telnet命令发送邮件
1. 基于ESMTP模式
2. 使用smtp.126.com服务器
3. AUTH LOGIN 使用加密的BASE64用户名密码，文中账号密码非真实有效账号

**高版本 Mac OS 的 telnet 因安全愿意已废弃，可使用 brew install telnet 安装**

```
C: telnet smtp.126.com 25    // 以telnet方式连接163邮件服务器
S: 220 126.com Anti-spam GT for Coremail System (126com[20140526])    // 220为响应数字，其后的为欢迎信息
C: helo smtp.126.com    // 除了HELO所具有的功能外，EHLO主要用来查询服务器支持的扩充功能
S: 250 OK
C: ehlo smtp.126.com
S: 250-mail
S: 250-PIPELINING
S: 250-AUTH LOGIN PLAIN
S: 250-AUTH=LOGIN PLAIN
S: 250-coremail 1Uxr2xKj7kG0xkI17xGrU7I0s8FY2U3Uj8Cz28x1UUUUU7Ic2I0Y2UFloyszUCa0xDrUUUUj
S: 250-STARTTLS
S: 250 8BITMIME
C: AUTH LOGIN    // 请求认证
S: 334 dXNlcm5hbWU6    // dXNlcm5hbWU6 为 username: 的base64编码
C: Y29zdGFAYW1heGl0Lm5ldA==    //发送经过BASE64编码的用户名
S: 334 UGFzc3dvcmQ6    // UGFzc3dvcmQ6 为 Password: 的base64编码
C: MTk4MjIxNA==  // 客户端发送的经过BASE64编码了的密码
S: 235 Authentication successful    // 认证成功
C: MAIL FROM: <hyy126126@126.com>    // 发送者邮箱  
S: 250 Mail OK
C: RCPT TO: <hyy126126@126.com>    // 接收者邮箱，包含所有的TO、CC、BCC
S: 250 Mail OK
C: DATA   // 请求发送数据  
S: 354 End data with <CR><LF>.<CR><LF> // 注意使用<CR><LF>.<CR><LF>作为结尾
C: FROM: <hyy126126@126.com> // 邮件需注意 from to subject格式否则可能出现空邮件
C: TO: <hyy126126@126.com>
C: SUBJECT: test
C: 
C: I am 1018ji!
C: 
C: .
S: 250 Mail OK queued as smtp1,C8mowABH0+D0kgZaV4mWAw--.35243S2 1510380604  
C: quit    //退出连接
S: 221 Bye
```
安装上述步骤就可以在邮件中接收到刚才发送的邮件了。
![mail_get](http://oz8myse7t.bkt.clouddn.com/0o0/text/1/mail_get.png)

状态码，摘自[SMTP状态码](http://www.codeweblog.com/smtp%25E7%258A%25B6%25E6%2580%2581%25E7%25A0%2581/)

| 状态码 | 解释 |
| --- | --- |
| 500 | 格式错误，命令不可识别（此错误也包括命令行过长） |
| 501 | 参数格式错误 |
| 502 | 命令不可实现 |
| 503 | 错误的命令序列 |
| 504 | 命令参数不可实现 |
| 211 | 系统状态或系统帮助响应 |
| 214 | 帮助信息 |
| 220 | 服务就绪 |
| 221 | 服务关闭传输信道 |
| 421 | 服务未就绪，关闭传输信道（当必须关闭时，此应答可以作为对任何命令的响应） |
| 250 | 要求的邮件操作完成 |
| 251 | 用户非本地，将转发向 |
| 450 | 要求的邮件操作未完成，邮箱不可用（例如，邮箱忙） |
| 550 | 要求的邮件操作未完成，邮箱不可用（例如，邮箱未找到，或不可访问） |
| 451 | 放弃要求的操作；处理过程中出错 |
| 551 | 用户非本地，请尝试 |
| 452 | 系统存储不足，要求的操作未执行 |
| 552 | 过量的存储分配，要求的操作未执行 |
| 553 | 邮箱名不可用，要求的操作未执行（例如邮箱格式错误） |
| 354 | 开始邮件输入，以.结束 |
| 554 | 操作失败 |
| 535 | 用户验证失败 |
| 235 | 用户验证成功 |
| 334 | 等待用户输入验证信息 |

SMTP常用命令
| 命令 | 解释 |
| --- | --- |
| ehlo<SP><domain><CRLF> | SMTP邮件发送程序与SMTP邮件接收程序建立连接后第一条发送命令，参数<domain>表示SMTP邮件发送者的主机名 |
| ehlo | 命令用于替代传统SMTP协议中的helo命令 |
| auth<SP><para><CRLF> | 如果SMTP邮件接收程序需要SMTP邮件发送程序进行认证时，它会向SMTP邮件发送程序提示它所采用的认证方式，SMTP邮件发送程序接着应该使用这个命令回应SMTP邮件接收程序，参数<para>表示回应的认证方式，通常是SMTP邮件接收程序先前提示的认证方式 |
| mail<SP>From:<reverse-path><CRLF> | 此命令用于指定邮件发送者的邮箱地址，参数<reverse-path>表示发件人的邮箱地址 |
| rcpt<SP>To:<forword-path><CRLF> | 此命令用于指定邮件接收者的邮箱地址，参数<forward-path>表示接收者的邮箱地址。如果邮件要发送给多个接收者，那么应使用多条rcpt<SP>To命令来分别指定每一个接收者的邮箱地址 |
| data<CRLF> | 此命令用于表示SMTP邮件发送程序准备开始输入邮件内容，在这个命令后面发送的所有数据都将被当做邮件内容，直至遇到“<CRLF>.<CRLF>"标志符，则表示邮件内容结束 |
| quit<CRLF> | 此命令表示要结束邮件发送过程，SMTP邮件接收程序接收到此命令后，将关闭与SMTP邮件发送程序的网络连接 |

具体命令可参考
1. SMTP  [RFC821](http://www.faqs.org/rfcs/rfc821.html)
2. ESMTP [RFC1869](http://www.faqs.org/rfcs/rfc1869.html)
3. 其他RFC文档 [Email相关RFC](http://www.360doc.com/content/13/0116/10/3200886_260466636.shtml)

## 关于TO、CC、BCC

* TO 收件人
* CC 即 carbon copy 抄送
* BCC 即 blind carbon copy 暗送

简单解释就是邮件发送到到A，再抄送到B，再暗送到C。那么A、B、C都会收到这封邮件，并且内容一样。但是A、B能看到收件人是A并抄送给了B，但是它们（A、B）都不知道C也收到了这封邮件。

但邮件真是先发送到到A，再抄送到B，再暗送到C吗？

例如，发送一封邮件：
FROM: from_user@qudian.com
TO: to_user@qudian.com
CC: cc_user@qudian.com,
BCC: bcc_user@qudian.com
这封邮件要发送出是这样子的。

```
MAIL FROM: <from_user@qudian.com>
RCPT TO: <to_user@qudian.com>
RCPT TO: <cc_user@qudian.com>
RCPT TO: <bcc_user@qudian.com>
DATA
FROM: <from_user@qudian.com>
TO: <to_user@qudian.com>
SUBJECT: test
I am 1018ji!
.
```
**那显而易见的是邮件的DATA中并不包含BCC，但是与SMTP协商之时却告诉服务器邮件要发送给TO、CC、BCC，这就是为什么TO、CC看不见BCC的原因！666！**

## 一个问题？
假定一个场景，一封邮件发送给
TO: to_user@qudian.com
BCC: bcc_group@qudian.com
**bcc_group@qudian.com作为一个邮件组包含  to_user@qudian.com 与 to_user_other@qudian.com**
那么 to_user@qudian.com、to_user_other@qudian.com会收到几封邮件，测试结果如下：

|  | to_user@qudian.com | to_user_other@qudian.com |
|---|---|---|
| WEB发送 | 1 | 1 |
| Swift Mailer 5.4.8 | 2 | 1 |
| PHPMailer 5.2.23 | 1 | 1 |

Swift Mailer 对于BCC的实现为先发送TO、CC一封邮件，然后发送BCC一封邮件，但其余的为同时发送TO、CC、BCC一封邮件。

************

**THE END**


