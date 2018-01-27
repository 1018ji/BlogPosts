---
title: Laravel 源码分析第一篇 Facades
date: 2018-01-24 13:16:13
tags:
---

Laravel 源码分析第一篇 Facades
===

# Laravel Facades 使用

**基于 Laravel 5.3.31 分析**

## 使用方法

参照：[Facades 使用文档](http://laravelacademy.org/post/6718.html)

---------------------

# Laravel Facades 启动加载过程

## Facades 源码入口

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

路径：Illuminate/Foundation/Console/Kernel.php

```php
public function handle($input, $output = null)
{
    try {
        # make 实例化 $bootstrappers 的配置
        # 调用 各个实例中的 bootstrap 方法加载 bootstrappers 变量
        # 门面 对应的类为 RegisterFacades
        # 门面类 路径 Illuminate/Foundation/Bootstrap/RegisterFacades.php
        $this->bootstrap();

        if (! $this->commandsLoaded) {
            $this->commands();

            $this->commandsLoaded = true;
        }

        return $this->getArtisan()->run($input, $output);
    } catch (Exception $e) {
        $this->reportException($e);

        $this->renderException($output, $e);

        return 1;
    } catch (Throwable $e) {
        $e = new FatalThrowableError($e);

        $this->reportException($e);

        $this->renderException($output, $e);

        return 1;
    }
}
```

1. 以上使用的为 Console 的 Kernal 类，Http 的 Kernal 类位置为 Illuminate/Foundation/Http/Kernel.php，同样适用 handle 方法调用 sendRequestThroughRouter 方法进而调用 bootstrap 方法，从而加载 bootstrappers 变量

## RegisterFacades 类

**重点查看调用的 bootstrap 方法**
路径：Illuminate/Foundation/Bootstrap/RegisterFacades.php

```php
public function bootstrap(Application $app)
{
    # 清理 Facade 实例的 resolvedInstance 变量
    Facade::clearResolvedInstances();

    # 将 Application 实例注入 Facade 实例，set 注入方法
    Facade::setFacadeApplication($app);

    # 获取 AliasLoader 实例注册别名
    # 配置为 config/app.php 下的 aliases 数组
    # 调用 AliasLoader load 方法，实际调用 PHP 底层 class_alias 注册别名
    AliasLoader::getInstance($app->make('config')->get('app.aliases', []))->register();
}
```
## AliasLoader类

路径：Illuminate/Foundation/AliasLoader.php
**以下为调用的三个方法**

```php
# 将别名注入到 AliasLoader 实例的 aliases 变量
public static function getInstance(array $aliases = [])
{
    if (is_null(static::$instance)) {
        return static::$instance = new static($aliases);
    }

    $aliases = array_merge(static::$instance->getAliases(), $aliases);

    static::$instance->setAliases($aliases);

    return static::$instance;
}
```

```php
public function register()
{
    if (! $this->registered) {
        $this->prependToLoaderStack();

        $this->registered = true;
    }
}

# 将 load 方法注册为 自动加载函数
protected function prependToLoaderStack()
{
    spl_autoload_register([$this, 'load'], true, true);
}

# 这个只在实际调用的时候运行的自动加载函数
public function load($alias)
{
    if (isset($this->aliases[$alias])) {
        return class_alias($this->aliases[$alias], $alias);
    }
}
```
1. 当使用门面例如 DB 时，会调用 load 方法并将实际对应的门面类注册一个门面的别名，例如 Illuminate\Support\Facades\DB::class 注册别名为 DB，如果使用DB，实际使用的为 Illuminate\Support\Facades\DB::class。**门面对应的实际类会有其他的自动加载函数进行加载**。
2. 根据命名空间规则 \DB 会被解析为为 DB，建议直接使用 use DB 方式，namespace 转换规则为：[规则](http://php.net/manual/zh/language.namespaces.rules.php)，建议完整学习 [namespaces](http://php.net/manual/zh/language.namespaces.php)。
3. 链接为 [spl_autoload_register](http://www.php.net/manual/zh/function.spl-autoload-register.php) 使用方法。

---------------------

# Laravel Facades 代用过程

## Facades 使用代码

```php
use DB;

public function getFileListData()
{
    $sql = "SELECT
                list.id,
                list.t_unique_id,
                list.t_observe_date,
                list.t_config
            FROM t_report_list list
            JOIN t_task_queue task ON list.t_unique_id = task.t_unique_id AND task.t_status = 4
            LEFT JOIN t_data_file_log log ON list.id = log.t_id
            WHERE (list.type = 1 or list.type = 4) AND log.id IS NULL
            GROUP BY list.t_unique_id
            LIMIT 5
            ";

    return DB::select($sql);
}
```
1. 以下分析基于 DB 门面进行分析。
2. 当使用 DB 时，按照前部分分析，会调用 AliasLoader 类 load 方法，将类 Illuminate\Support\Facades\DB::class 注册别名 DB，故当使用 DB 时，实际使用的是 Illuminate\Support\Facades\DB::class。

## DB 类

路径：Illuminate/Support/Facades/DB.php

```php
class DB extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'db';
    }
}
```
1. DB 类集成 Facade，只实现 getFacadeAccessor 方法，具体的有效代码位于其父类 Facade。

## Facade 类

```php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}

public static function getFacadeRoot()
{
  return static::resolveFacadeInstance(static::getFacadeAccessor());
}

# name 为子类 getFacadeAccessor 方法的返回值，对于 DB 门面为 db
protected static function resolveFacadeInstance($name)
{
    # 如为对象直接返回
    if (is_object($name)) {
        return $name;
    }

    # 如加载过，则直接取第一次加载的值
    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    # 首次加载的方法
    return static::$resolvedInstance[$name] = static::$app[$name];
}
```
1. 当执行 DB::select 时，DB 类不含 select 方法，切为 static 调用方式，时机会调用 DB 父类中的 __callStatic 方法。
2. static::$app 为通过函数 setFacadeApplication 注入，Facade 实际为单例模式。
3. 变量 instance 实际为 static::$app['db']，但变量 app 为 Application 实例，这里要引入 ArrayAccess 实现对象数组访问，具体访问的为 Container 类的 offsetGet 方法。
4. [ArrayAccess 参考文档](http://php.net/manual/zh/class.arrayaccess.php)

```php
public function offsetGet($key)
{
    return $this->make($key);
}
```
1. make 方法会解析出真正使用的类，门面 DB 真正使用的 DatabaseServiceProvider 类的 register 方法获取 DatabaseManager 类的单例对象。
2. 对于 make 方法参考 [服务容器 文档](http://laravelacademy.org/post/5805.html)

## DatabaseServiceProvider 类

路径：Illuminate/Database/DatabaseServiceProvider.php

```php
public function register()
{
    Model::clearBootedModels();

    $this->registerEloquentFactory();

    $this->registerQueueableEntityResolver();

    // The connection factory is used to create the actual connection instances on
    // the database. We will inject the factory into the manager so that it may
    // make the connections while they are actually needed and not of before.
    $this->app->singleton('db.factory', function ($app) {
        return new ConnectionFactory($app);
    });

    // The database manager is used to resolve various connections, since multiple
    // connections might be managed. It also implements the connection resolver
    // interface which may be used by other components requiring connections.
    $this->app->singleton('db', function ($app) {
        return new DatabaseManager($app, $app['db.factory']);
    });

    $this->app->bind('db.connection', function ($app) {
        return $app['db']->connection();
    });
}
```


---------------------

**END**


 