---
title: Laravel 源码分析第二篇 Contracts
date: 2018-01-06 21:37:06
categories: Laravel
tags:
  - PHP
  - Laravel
---

# Laravel Contracts 使用

**基于 Laravel 5.3.31 分析**

## 使用方法

参照：[Contracts 使用文档](http://laravelacademy.org/post/5826.html)

---------------------

# Laravel Contracts 启动加载过程

## Contracts 源码入口

路径：public/index.php

```php
require __DIR__.'/../bootstrap/autoload.php';

/*
|--------------------------------------------------------------------------
| Turn On The Lights
|--------------------------------------------------------------------------
|
| We need to illuminate PHP development, so let us turn on the lights.
| This bootstraps the framework and gets it ready for use, then it
| will load up this application so that we can run it and send
| the responses back to the browser and delight our users.
|
*/

# 实例化 Illuminate\Foundation\Application
# Application 路径 Illuminate/Foundation/Application.php
$app = require_once __DIR__.'/../bootstrap/app.php';

/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
| Once we have the application, we can handle the incoming request
| through the kernel, and send the associated response back to
| the client's browser allowing them to enjoy the creative
| and wonderful application we have prepared for them.
|
*/

# 实例化 kernel 
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

# 处理捕获请求
# 查看源码实际执行 kernal 的 handle 方法
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

$response->send();

$kernel->terminate($request, $response);
```

## kernal 类

路径：Illuminate/Foundation/Http/Kernel.php

```php
protected $bootstrappers = [
    'Illuminate\Foundation\Bootstrap\DetectEnvironment',
    'Illuminate\Foundation\Bootstrap\LoadConfiguration',
    'Illuminate\Foundation\Bootstrap\ConfigureLogging',
    'Illuminate\Foundation\Bootstrap\HandleExceptions',
    'Illuminate\Foundation\Bootstrap\RegisterFacades',
    'Illuminate\Foundation\Bootstrap\RegisterProviders',
    'Illuminate\Foundation\Bootstrap\BootProviders',
];

public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    } catch (Throwable $e) {
        $this->reportException($e = new FatalThrowableError($e));

        $response = $this->renderException($request, $e);
    }

    $this->app['events']->fire('kernel.handled', [$request, $response]);

    return $response;
}

protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}

public function bootstrap()
{
    if (! $this->app->hasBeenBootstrapped()) {
        $this->app->bootstrapWith($this->bootstrappers());
    }
}

######################## 此后方法位于 Application ########################################
public function bootstrapWith(array $bootstrappers)
{
    $this->hasBeenBootstrapped = true;

    foreach ($bootstrappers as $bootstrapper) {
        $this['events']->fire('bootstrapping: '.$bootstrapper, [$this]);

        $this->make($bootstrapper)->bootstrap($this);

        $this['events']->fire('bootstrapped: '.$bootstrapper, [$this]);
    }
}

public function registerConfiguredProviders()
{
    $manifestPath = $this->getCachedServicesPath();

    (new ProviderRepository($this, new Filesystem, $manifestPath))
                ->load($this->config['app.providers']);
}

```
1. 按照以上代码分析当执行到 Illuminate\Foundation\Bootstrap\RegisterProviders 时候会注册所有的服务提供者，具体过程为执行 bootstrapWith 方法，获取 RegisterProviders 实例，执行 bootstrap 方法进而调用 Application 的 registerConfiguredProviders方法。
2. 举例说明对于服务提供者 BroadcastServiceProvider（Illuminate/Broadcasting/BroadcastServiceProvider.php） 的 register 方法中 Contracts 设置为使用 Illuminate\Contracts\Broadcasting\Broadcaster 调用 Illuminate\Broadcasting\BroadcastManager

```php
public function register()
{
    $this->app->singleton('Illuminate\Broadcasting\BroadcastManager', function ($app) {
        return new BroadcastManager($app);
    });

    $this->app->singleton('Illuminate\Contracts\Broadcasting\Broadcaster', function ($app) {
        return $app->make('Illuminate\Broadcasting\BroadcastManager')->connection();
    });

    $this->app->alias(
        'Illuminate\Broadcasting\BroadcastManager', 'Illuminate\Contracts\Broadcasting\Factory'
    );
}
```

# Laravel Contracts 使用

1. 使用接口注入
2. 使用服务提供者解析获取真正的类