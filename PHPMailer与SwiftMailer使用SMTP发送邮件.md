---
title: PHPMailer 与 SwiftMailer 使用 SMTP 发送邮件
date: 2018-01-24 13:16:13
tags:
---

PHPMailer 与 SwiftMailer 使用 SMTP 发送邮件
===

# PHPMailer

## 源码以及稳定版本：[GitHub](https://github.com/PHPMailer/PHPMailer)

1.  [PHPMailer 6.0.2](https://github.com/PHPMailer/PHPMailer/releases/tag/v6.0.2)
2.  [PHPMailer 5.2.26](https://github.com/PHPMailer/PHPMailer/releases/tag/v5.2.26)

此后分析将基于 5.2.26

## SMTP使用示例


```php
<?php
// Import PHPMailer classes into the global namespace
// These must be at the top of your script, not inside a function
use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\Exception;

//Load composer's autoloader
require 'vendor/autoload.php';

$mail = new PHPMailer(true);                              // Passing `true` enables exceptions
try {
 //Server settings
 $mail->SMTPDebug = 2;                                 // Enable verbose debug output
 $mail->isSMTP();                                      // Set mailer to use SMTP
 $mail->Host = 'smtp1.example.com;smtp2.example.com';  // Specify main and backup SMTP servers
 $mail->SMTPAuth = true;                               // Enable SMTP authentication
 $mail->Username = 'user@example.com';                 // SMTP username
 $mail->Password = 'secret';                           // SMTP password
 $mail->SMTPSecure = 'tls';                            // Enable TLS encryption, `ssl` also accepted
 $mail->Port = 587;                                    // TCP port to connect to

 //Recipients
 $mail->setFrom('from@example.com', 'Mailer');
 $mail->addAddress('joe@example.net', 'Joe User');     // Add a recipient
 $mail->addAddress('ellen@example.com');               // Name is optional
 $mail->addReplyTo('info@example.com', 'Information');
 $mail->addCC('cc@example.com');
 $mail->addBCC('bcc@example.com');

 //Attachments
 $mail->addAttachment('/var/tmp/file.tar.gz');         // Add attachments
 $mail->addAttachment('/tmp/image.jpg', 'new.jpg');    // Optional name

 //Content
 $mail->isHTML(true);                                  // Set email format to HTML
 $mail->Subject = 'Here is the subject';
 $mail->Body    = 'This is the HTML message body <b>in bold!</b>';
 $mail->AltBody = 'This is the body in plain text for non-HTML mail clients';

 $mail->send();
 echo 'Message has been sent';
} catch (Exception $e) {
 echo 'Message could not be sent.';
 echo 'Mailer Error: ' . $mail->ErrorInfo;
}
```
更多详见 [examples](https://github.com/PHPMailer/PHPMailer/tree/5.2-stable/examples "examples") 文件夹

## send函数

```php
public function send()
{
    try {
        if (!$this->preSend()) {
            return false;
        }
        return $this->postSend();
    } catch (phpmailerException $exc) {
        $this->mailHeader = '';
        $this->setError($exc->getMessage());
        if ($this->exceptions) {
            throw $exc;
        }
        return false;
    }
}
```
先执行 preSend 预发送函数，然后通过 postSend 函数发送邮件

## postSend函数


```php
public function postSend()
{
    try {
        // Choose the mailer and send through it
        switch ($this->Mailer) {
            case 'sendmail':
            case 'qmail':
                return $this->sendmailSend($this->MIMEHeader, $this->MIMEBody);
            case 'smtp':
                return $this->smtpSend($this->MIMEHeader, $this->MIMEBody);
            case 'mail':
                return $this->mailSend($this->MIMEHeader, $this->MIMEBody);
            default:
                $sendMethod = $this->Mailer.'Send';
                if (method_exists($this, $sendMethod)) {
                    return $this->$sendMethod($this->MIMEHeader, $this->MIMEBody);
                }

                return $this->mailSend($this->MIMEHeader, $this->MIMEBody);
        }
    } catch (phpmailerException $exc) {
        $this->setError($exc->getMessage());
        $this->edebug($exc->getMessage());
        if ($this->exceptions) {
            throw $exc;
        }
    }
    return false;
}
```
很容易看出 SMTP 发送邮件使用的为 smtpSend 函数

## smtpSend函数

```php
protected function smtpSend($header, $body)
{
    $bad_rcpt = array();
    if (!$this->smtpConnect($this->SMTPOptions)) {
        throw new phpmailerException($this->lang('smtp_connect_failed'), self::STOP_CRITICAL);
    }
    if (!empty($this->Sender) and $this->validateAddress($this->Sender)) {
        $smtp_from = $this->Sender;
    } else {
        $smtp_from = $this->From;
    }
    if (!$this->smtp->mail($smtp_from)) {
        $this->setError($this->lang('from_failed') . $smtp_from . ' : ' . implode(',', $this->smtp->getError()));
        throw new phpmailerException($this->ErrorInfo, self::STOP_CRITICAL);
    }

    // Attempt to send to all recipients
    foreach (array($this->to, $this->cc, $this->bcc) as $togroup) {
        foreach ($togroup as $to) {
            if (!$this->smtp->recipient($to[0])) {
                $error = $this->smtp->getError();
                $bad_rcpt[] = array('to' => $to[0], 'error' => $error['detail']);
                $isSent = false;
            } else {
                $isSent = true;
            }
            $this->doCallback($isSent, array($to[0]), array(), array(), $this->Subject, $body, $this->From);
        }
    }

    // Only send the DATA command if we have viable recipients
    if ((count($this->all_recipients) > count($bad_rcpt)) and !$this->smtp->data($header . $body)) {
        throw new phpmailerException($this->lang('data_not_accepted'), self::STOP_CRITICAL);
    }
    if ($this->SMTPKeepAlive) {
        $this->smtp->reset();
    } else {
        $this->smtp->quit();
        $this->smtp->close();
    }
    //Create error message for any bad addresses
    if (count($bad_rcpt) > 0) {
        $errstr = '';
        foreach ($bad_rcpt as $bad) {
            $errstr .= $bad['to'] . ': ' . $bad['error'];
        }
        throw new phpmailerException(
            $this->lang('recipients_failed') . $errstr,
            self::STOP_CONTINUE
        );
    }
    return true;
}
```
1. all recipients 都会通过 SMTP 类 recipient 函数发送 RCPT TO 命令
如失败则 isSent 置为 false，而且收件人会计入 bad_rcpt
2. doCallback 为回调函数，在 send 时候会被调用，可以通过 set 函数设置
3. all_recipients > bad_rcpt 并且 SMTP 类 data 函数执行成功（表明发送 DATA 命令成功），否则抛出异常
4. bad_rcpt 如果不为空，则会抛出一个异常，但**不保证所有的邮件没有发出**

6.0.2版本 smtpSend 函数

```php
protected function smtpSend($header, $body)
{
    $bad_rcpt = [];
    if (!$this->smtpConnect($this->SMTPOptions)) {
        throw new Exception($this->lang('smtp_connect_failed'), self::STOP_CRITICAL);
    }
    //Sender already validated in preSend()
    if ('' == $this->Sender) {
        $smtp_from = $this->From;
    } else {
        $smtp_from = $this->Sender;
    }
    if (!$this->smtp->mail($smtp_from)) {
        $this->setError($this->lang('from_failed') . $smtp_from . ' : ' . implode(',', $this->smtp->getError()));
        throw new Exception($this->ErrorInfo, self::STOP_CRITICAL);
    }

    $callbacks = [];
    // Attempt to send to all recipients
    foreach ([$this->to, $this->cc, $this->bcc] as $togroup) {
        foreach ($togroup as $to) {
            if (!$this->smtp->recipient($to[0])) {
                $error = $this->smtp->getError();
                $bad_rcpt[] = ['to' => $to[0], 'error' => $error['detail']];
                $isSent = false;
            } else {
                $isSent = true;
            }

            $callbacks[] = ['issent'=>$isSent, 'to'=>$to[0]];
        }
    }

    // Only send the DATA command if we have viable recipients
    if ((count($this->all_recipients) > count($bad_rcpt)) and !$this->smtp->data($header . $body)) {
        throw new Exception($this->lang('data_not_accepted'), self::STOP_CRITICAL);
    }

    $smtp_transaction_id = $this->smtp->getLastTransactionID();

    if ($this->SMTPKeepAlive) {
        $this->smtp->reset();
    } else {
        $this->smtp->quit();
        $this->smtp->close();
    }

    foreach ($callbacks as $cb) {
        $this->doCallback(
            $cb['issent'],
            [$cb['to']],
            [],
            [],
            $this->Subject,
            $body,
            $this->From,
            ['smtp_transaction_id' => $smtp_transaction_id]
        );
    }

    //Create error message for any bad addresses
    if (count($bad_rcpt) > 0) {
        $errstr = '';
        foreach ($bad_rcpt as $bad) {
            $errstr .= $bad['to'] . ': ' . $bad['error'];
        }
        throw new Exception(
            $this->lang('recipients_failed') . $errstr,
            self::STOP_CONTINUE
        );
    }

    return true;
}
```
1. doCallback 移到 this->smtp->data 发送数据之后
为处理 **SMTP transaction id** 后移

## SMTP 类 data 函数

**实际执行 SMTP 的 DATA 命令发送数据**


```
public function data($msg_data)
{
    //This will use the standard timelimit
    if (!$this->sendCommand('DATA', 'DATA', 354)) {
        return false;
    }

    /* The server is ready to accept data!
     * According to rfc821 we should not send more than 1000 characters on a single line (including the CRLF)
     * so we will break the data up into lines by \r and/or \n then if needed we will break each of those into
     * smaller lines to fit within the limit.
     * We will also look for lines that start with a '.' and prepend an additional '.'.
     * NOTE: this does not count towards line-length limit.
     */

    // Normalize line breaks before exploding
    $lines = explode("\n", str_replace(array("\r\n", "\r"), "\n", $msg_data));

    /* To distinguish between a complete RFC822 message and a plain message body, we check if the first field
     * of the first line (':' separated) does not contain a space then it _should_ be a header and we will
     * process all lines before a blank line as headers.
     */

    $field = substr($lines[0], 0, strpos($lines[0], ':'));
    $in_headers = false;
    if (!empty($field) && strpos($field, ' ') === false) {
        $in_headers = true;
    }

    foreach ($lines as $line) {
        $lines_out = array();
        if ($in_headers and $line == '') {
            $in_headers = false;
        }
        //Break this line up into several smaller lines if it's too long
        //Micro-optimisation: isset($str[$len]) is faster than (strlen($str) > $len),
        while (isset($line[self::MAX_LINE_LENGTH])) {
            //Working backwards, try to find a space within the last MAX_LINE_LENGTH chars of the line to break on
            //so as to avoid breaking in the middle of a word
            $pos = strrpos(substr($line, 0, self::MAX_LINE_LENGTH), ' ');
            //Deliberately matches both false and 0
            if (!$pos) {
                //No nice break found, add a hard break
                $pos = self::MAX_LINE_LENGTH - 1;
                $lines_out[] = substr($line, 0, $pos);
                $line = substr($line, $pos);
            } else {
                //Break at the found point
                $lines_out[] = substr($line, 0, $pos);
                //Move along by the amount we dealt with
                $line = substr($line, $pos + 1);
            }
            //If processing headers add a LWSP-char to the front of new line RFC822 section 3.1.1
            if ($in_headers) {
                $line = "\t" . $line;
            }
        }
        $lines_out[] = $line;

        //Send the lines to the server
        foreach ($lines_out as $line_out) {
            //RFC2821 section 4.5.2
            if (!empty($line_out) and $line_out[0] == '.') {
                $line_out = '.' . $line_out;
            }
            $this->client_send($line_out . self::CRLF);
        }
    }

    //Message data has been sent, complete the command
    //Increase timelimit for end of DATA command
    $savetimelimit = $this->Timelimit;
    $this->Timelimit = $this->Timelimit * 2;
    $result = $this->sendCommand('DATA END', '.', 250);
    $this->recordLastTransactionID();
    //Restore timelimit
    $this->Timelimit = $savetimelimit;
    return $result;
}
```
1. 此函数会根据 MAX_LINE_LENGTH = 998（尾部的 /r/n 最多占去 2 个字符，实际单行最大 1000）
2. 使用 <CR><LF>.<CR><LF> 结束 **DATA** 命令

附表
**各系统对于 /r/n 的使用**

| 系统 | 行尾附加中文 | 行尾附件英文 |
|---|---|---|
| Windows | **<换行><回车>** | \r\n |
| Unix、Linux | **<回车>** | \n |
| Mac | **<换行>** | \r |

-----

# Swift Mailer

## 源码以及稳定版本：[GitHub](https://github.com/swiftmailer/swiftmailer)

1.  [PHPMailer 6.0.2](https://github.com/swiftmailer/swiftmailer/releases/tag/v6.0.2)
2.  [PHPMailer 5.4.8](https://github.com/swiftmailer/swiftmailer/releases/tag/v5.4.8)

此后分析将基于 5.4.8

## SMTP使用示例


```php
// Create the Transport
$transport = (new Swift_SmtpTransport('smtp.example.org', 25))
  ->setUsername('your username')
  ->setPassword('your password')
;

// Create the Mailer using your created Transport
$mailer = new Swift_Mailer($transport);

// Create a message
$message = (new Swift_Message('Wonderful Subject'))
  ->setFrom(['john@doe.com' => 'John Doe'])
  ->setTo(['receiver@domain.org', 'other@domain.org' => 'A name'])
  ->setBody('Here is the message itself')
  ;

// Send the message
$result = $mailer->send($message);
```
## send 函数


```php
public function send(Swift_Mime_Message $message, &$failedRecipients = null)
{
    $failedRecipients = (array) $failedRecipients;

    if (!$this->_transport->isStarted()) {
        $this->_transport->start();
    }

    $sent = 0;

    try {
        $sent = $this->_transport->send($message, $failedRecipients);
    } catch (Swift_RfcComplianceException $e) {
        foreach ($message->getTo() as $address => $name) {
            $failedRecipients[] = $address;
        }
    }

    return $sent;
}
```
1. SMTP 发送邮件使用的是 Swift_SmtpTransport，通过构造函数注入，注入变量为 $this->_transport
2. 实际发送调用 Swift_SmtpTransport 的 send 方法

## Swift_SmtpTransport 的 send 函数

**实际调用的是抽象类 Swift_Transport_AbstractSmtpTransport 的 send 方法**


```php
public function send(Swift_Mime_Message $message, &$failedRecipients = null)
{
    $sent = 0;
    $failedRecipients = (array) $failedRecipients;

    if ($evt = $this->_eventDispatcher->createSendEvent($this, $message)) {
        $this->_eventDispatcher->dispatchEvent($evt, 'beforeSendPerformed');
        if ($evt->bubbleCancelled()) {
            return 0;
        }
    }

    if (!$reversePath = $this->_getReversePath($message)) {
        $this->_throwException(new Swift_TransportException(
            'Cannot send message without a sender address'
            )
        );
    }

    $to = (array) $message->getTo();
    $cc = (array) $message->getCc();
    $tos = array_merge($to, $cc);
    $bcc = (array) $message->getBcc();

    $message->setBcc(array());

    try {
        $sent += $this->_sendTo($message, $reversePath, $tos, $failedRecipients);
        $sent += $this->_sendBcc($message, $reversePath, $bcc, $failedRecipients);
    } catch (Exception $e) {
        $message->setBcc($bcc);
        throw $e;
    }

    $message->setBcc($bcc);

    if ($evt) {
        if ($sent == count($to) + count($cc) + count($bcc)) {
            $evt->setResult(Swift_Events_SendEvent::RESULT_SUCCESS);
        } elseif ($sent > 0) {
            $evt->setResult(Swift_Events_SendEvent::RESULT_TENTATIVE);
        } else {
            $evt->setResult(Swift_Events_SendEvent::RESULT_FAILED);
        }
        $evt->setFailedRecipients($failedRecipients);
        $this->_eventDispatcher->dispatchEvent($evt, 'sendPerformed');
    }

    $message->generateId(); //Make sure a new Message ID is used

    return $sent;
}
```

```php
$sent += $this->_sendTo($message, $reversePath, $tos, $failedRecipients);
$sent += $this->_sendBcc($message, $reversePath, $bcc, $failedRecipients);
```
1. 以上代码为实际发送邮件的代码，首先发送 TO（包含 TO、CC 收件人），然后发送 BCC
2. send 函数返回执行 _sendTo 与 _sendBcc 的实际收件人的个数

## _sendTo 函数


```php
private function _sendTo(Swift_Mime_Message $message, $reversePath, array $to, array &$failedRecipients)
{
    if (empty($to)) {
        return 0;
    }

    return $this->_doMailTransaction($message, $reversePath, array_keys($to),
        $failedRecipients);
}
```
实际执行 _doMailTransaction 函数


```php
private function _doMailTransaction($message, $reversePath, array $recipients, array &$failedRecipients)
{
    $sent = 0;
    $this->_doMailFromCommand($reversePath);
    foreach ($recipients as $forwardPath) {
        try {
            $this->_doRcptToCommand($forwardPath);
            ++$sent;
        } catch (Swift_TransportException $e) {
            $failedRecipients[] = $forwardPath;
        }
    }

    if ($sent != 0) {
        $this->_doDataCommand();
        $this->_streamMessage($message);
    } else {
        $this->reset();
    }

    return $sent;
}
```
1. 执行 **MAIL FROM** 命令
2. 执行 **RCPT TO** 命令，成功则 sent 加 1，否则计入 failedRecipients
3. 如果 sent != 0，则执行 **DATA** 命令，再次执行邮件数据写入，最后发送 **\r\n.\r\n** 结束 DATA 命令（此时邮件已被发送出去）。
4. 如果 sent = 0，则使用 **RSET** 命令重置回话

## _streamMessage 函数

**实际执行的发送邮件的操作**
**_doDataCommand 实际只执行 DATA 命令，与 SMTP 服务器确认是否可以发送数据**

```
protected function _streamMessage(Swift_Mime_Message $message)
{
    $this->_buffer->setWriteTranslations(array("\r\n." => "\r\n.."));
    try {
        $message->toByteStream($this->_buffer);
        $this->_buffer->flushBuffers();
    } catch (Swift_TransportException $e) {
        $this->_throwException($e);
    }
    $this->_buffer->setWriteTranslations(array());
    $this->executeCommand("\r\n.\r\n", array(250));
}
```


## _sendBcc 函数

```php
private function _sendBcc(Swift_Mime_Message $message, $reversePath, array $bcc, array &$failedRecipients)
{
    $sent = 0;
    foreach ($bcc as $forwardPath => $name) {
        $message->setBcc(array($forwardPath => $name));
        $sent += $this->_doMailTransaction(
            $message, $reversePath, array($forwardPath), $failedRecipients
            );
    }

    return $sent;
}
```
1. 每次执行 setBcc 操作
2. 对于 BCC，单独发送邮件拷贝
3. 注意 **TO CC 为同一时间发送的邮件**

--------------

**THE END**

