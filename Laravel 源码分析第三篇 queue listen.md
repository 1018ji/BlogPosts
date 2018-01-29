---
title: Laravel 源码分析第三篇 queue listen
date: 2018-01-28 19:47:06
categories: Laravel
tags:
  - PHP
  - Laravel
---

# Laravel Queue 使用

**基于 Lumen 5.5.2 分析（Laravel 可具体自行分析）**

## 使用方法

参照：[Contracts 使用文档](https://d.laravel-china.org/docs/5.5/queues)

---------------------

# Laravel Queue listen

## queue:listen 源码入口

路径：laravel/framework/src/Illuminate/Queue/Console/ListenCommand.php

```php
public function __construct(Listener $listener)
{
    parent::__construct();

    $this->setOutputHandler($this->listener = $listener);
}

public function handle()
{
    // We need to get the right queue for the connection which is set in the queue
    // configuration file for the application. We will pull it based on the set
    // connection being run for the queue operation currently being executed.
    $queue = $this->getQueue(
        $connection = $this->input->getArgument('connection')
    );

    # 调用 Listener 的 listen 函数，执行 queue:work 消费队列
    $this->listener->listen(
        $connection, $queue, $this->gatherOptions()
    );
}
```
1. 注入 Listener 类
2. queue 为 artisan 脚本，handle 为实际执行函数

<!-- more -->

## Listener 类

路径：laravel/framework/src/Illuminate/Queue/Listener.php

```php
public function listen($connection, $queue, ListenerOptions $options)
{
    $process = $this->makeProcess($connection, $queue, $options);

    while (true) {
        $this->runProcess($process, $options->memory);
    }
}
```
1. 首先根据参数构建 process，死循环运行 runProcess 消耗队列
2. 调用 Process 类的 runProcess 方法处理当前的 proces，后续详细分析

```php
public function makeProcess($connection, $queue, ListenerOptions $options)
{
    $command = $this->workerCommand;

    // If the environment is set, we will append it to the command string so the
    // workers will run under the specified environment. Otherwise, they will
    // just run under the production environment which is not always right.
    if (isset($options->environment)) {
        $command = $this->addEnvironment($command, $options);
    }

    // Next, we will just format out the worker commands with all of the various
    // options available for the command. This will produce the final command
    // line that we will pass into a Symfony process object for processing.
    $command = $this->formatCommand(
        $command, $connection, $queue, $options
    );

    return new Process(
        $command, $this->commandPath, null, null, $options->timeout
    );
}
```
1. $this->workerCommand 在构造函数执行 buildCommandTemplate 函数赋值，为执行的 queue:work 的模板命令，即：queue:work %s --once --queue=%s --delay=%s --memory=%s --sleep=%s --tries=%s
2. addEnvironment 函数处理附加的环境变量参数
3. formatCommand 函数根据具体配置参数构建实际执行的命令（queue:work），但是该命令携带 --once 参数只会执行一次，故需要死循环持续运行，这也就是为什么 queue:listen 为什么不需要重启的原因
4. 调用 Process 实际执行构造的命令

```php
public function runProcess(Process $process, $memory)
{
    $process->run(function ($type, $line) {
        $this->handleWorkerOutput($type, $line);
    });

    // Once we have run the job we'll go check if the memory limit has been exceeded
    // for the script. If it has, we will kill this script so the process manager
    // will restart this with a clean slate of memory automatically on exiting.
    if ($this->memoryExceeded($memory)) {
        $this->stop();
    }
}
```
1. 调用 Process 类的 run 方法执行上述构造的命令，其 callable 参数为输出执行过程中日志
2. runProcess 会检查内存消耗，如果超过配置内存限制将会使用 die 结束进程

## Process 类

路径：symfony/process/Process.php

**此类属于 symfony 组件，该类为 PHP proc_* 类似方法的包装类**

```php
public function run($callback = null/*, array $env = array()*/)
{
    $env = 1 < func_num_args() ? func_get_arg(1) : null;
    $this->start($callback, $env);

    return $this->wait();
}
```
1. start 实际执行 queue:work 方法
2. wait 等待 queue:work 执行完成

```php
public function start(callable $callback = null/*, array $env = array()*/)
{
    if ($this->isRunning()) {
        throw new RuntimeException('Process is already running');
    }
    if (2 <= func_num_args()) {
        $env = func_get_arg(1);
    } else {
        if (__CLASS__ !== static::class) {
            $r = new \ReflectionMethod($this, __FUNCTION__);
            if (__CLASS__ !== $r->getDeclaringClass()->getName() && (2 > $r->getNumberOfParameters() || 'env' !== $r->getParameters()[0]->name)) {
                @trigger_error(sprintf('The %s::start() method expects a second "$env" argument since Symfony 3.3. It will be made mandatory in 4.0.', static::class), E_USER_DEPRECATED);
            }
        }
        $env = null;
    }

    $this->resetProcessData();
    $this->starttime = $this->lastOutputTime = microtime(true);
    $this->callback = $this->buildCallback($callback);
    $this->hasCallback = null !== $callback;
    $descriptors = $this->getDescriptors();
    $inheritEnv = $this->inheritEnv;

    if (is_array($commandline = $this->commandline)) {
        $commandline = implode(' ', array_map(array($this, 'escapeArgument'), $commandline));

        if ('\\' !== DIRECTORY_SEPARATOR) {
            // exec is mandatory to deal with sending a signal to the process
            $commandline = 'exec '.$commandline;
        }
    }

    if (null === $env) {
        $env = $this->env;
    } else {
        if ($this->env) {
            $env += $this->env;
        }
        $inheritEnv = true;
    }

    if (null !== $env && $inheritEnv) {
        $env += $this->getDefaultEnv();
    } elseif (null !== $env) {
        @trigger_error('Not inheriting environment variables is deprecated since Symfony 3.3 and will always happen in 4.0. Set "Process::inheritEnvironmentVariables()" to true instead.', E_USER_DEPRECATED);
    } else {
        $env = $this->getDefaultEnv();
    }
    if ('\\' === DIRECTORY_SEPARATOR && $this->enhanceWindowsCompatibility) {
        $this->options['bypass_shell'] = true;
        $commandline = $this->prepareWindowsCommandLine($commandline, $env);
    } elseif (!$this->useFileHandles && $this->enhanceSigchildCompatibility && $this->isSigchildEnabled()) {
        // last exit code is output on the fourth pipe and caught to work around --enable-sigchild
        $descriptors[3] = array('pipe', 'w');

        // See https://unix.stackexchange.com/questions/71205/background-process-pipe-input
        $commandline = '{ ('.$commandline.') <&3 3<&- 3>/dev/null & } 3<&0;';
        $commandline .= 'pid=$!; echo $pid >&3; wait $pid; code=$?; echo $code >&3; exit $code';

        // Workaround for the bug, when PTS functionality is enabled.
        // @see : https://bugs.php.net/69442
        $ptsWorkaround = fopen(__FILE__, 'r');
    }
    if (defined('HHVM_VERSION')) {
        $envPairs = $env;
    } else {
        $envPairs = array();
        foreach ($env as $k => $v) {
            $envPairs[] = $k.'='.$v;
        }
    }

    if (!is_dir($this->cwd)) {
        @trigger_error('The provided cwd does not exist. Command is currently ran against getcwd(). This behavior is deprecated since Symfony 3.4 and will be removed in 4.0.', E_USER_DEPRECATED);
    }

    $this->process = proc_open($commandline, $descriptors, $this->processPipes->pipes, $this->cwd, $envPairs, $this->options);

    if (!is_resource($this->process)) {
        throw new RuntimeException('Unable to launch a new process.');
    }
    $this->status = self::STATUS_STARTED;

    if (isset($descriptors[3])) {
        $this->fallbackStatus['pid'] = (int) fgets($this->processPipes->pipes[3]);
    }

    if ($this->tty) {
        return;
    }

    $this->updateStatus(false);
    $this->checkTimeout();
}
```
1. 执行 proc_open 运行 queue:work 进程
2. 使用 updateStatus 方法更新 process 状态，并读取管道信息
3. 使用 checkTimeout 方法检查 process 是否超时，默认：60s
**此方法需要后续详细分析**

```php
public function wait(callable $callback = null)
{
    $this->requireProcessIsStarted(__FUNCTION__);

    $this->updateStatus(false);

    if (null !== $callback) {
        if (!$this->processPipes->haveReadSupport()) {
            $this->stop(0);
            throw new \LogicException('Pass the callback to the Process::start method or enableOutput to use a callback with Process::wait');
        }
        $this->callback = $this->buildCallback($callback);
    }

    do {
        $this->checkTimeout();
        $running = '\\' === DIRECTORY_SEPARATOR ? $this->isRunning() : $this->processPipes->areOpen();
        $this->readPipes($running, '\\' !== DIRECTORY_SEPARATOR || !$running);
    } while ($running);

    while ($this->isRunning()) {
        usleep(1000);
    }

    if ($this->processInformation['signaled'] && $this->processInformation['termsig'] !== $this->latestSignal) {
        throw new RuntimeException(sprintf('The process has been signaled with signal "%s".', $this->processInformation['termsig']));
    }

    return $this->exitcode;
}
```
1. 使用 requireProcessIsStarted 方法判断进程状态
2. 使用 updateStatus 方法更新 process 状态，并读取管道信息
3. 使用 checkTimeout 持续检查是否超时
**此方法需要后续详细分析**

---------------------

**END**
