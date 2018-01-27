# Laravel Session——session 的启动与运行源码分析

## 前言
在网页开发中， `session` 具有重要的作用，它可以在多个请求中存储用户的信息，用于识别用户的身份信息。`laravel` 为用户提供了可读性强的 API 处理各种自带的 Session 后台驱动程序。支持诸如比较热门的 Memcached、Redis 和开箱即用的数据库等常见的后台驱动程序。本文将会在本篇文章中讲述最常见的由 `File` 与 `redis` 驱动的 `session` 源码。

## session 服务的注册

与其他功能一样，`session` 由自己的服务提供者在 `container` 内进行注册：

```php

class SessionServiceProvider extends ServiceProvider 
{
    public function register()
    {
        $this->registerSessionManager();

        $this->registerSessionDriver();

        $this->app->singleton(StartSession::class);
    }
    
    protected function registerSessionManager()
    {
        $this->app->singleton('session', function ($app) {
            return new SessionManager($app);
        });
    }
    
    protected function registerSessionDriver()
    {
        $this->app->singleton('session.store', function ($app) {
            return $app->make('session')->driver();
        });
    }

}
```

可以看到 `SessionManager` 是整个 `session` 服务的接口类，一切对 `session` 的操作都是由这个类实现。`session.store` 是 `session` 服务的存储驱动。

## session 服务的启动

`session` 服务是以中间件的形式启动的，其中间件是 `Illuminate\Session\Middleware\StartSession`:

```php
public function handle($request, Closure $next)
{
    $this->sessionHandled = true;

    if ($this->sessionConfigured()) {
        $request->setLaravelSession(
            $session = $this->startSession($request)
        );

        $this->collectGarbage($session);
    }

    $response = $next($request);

    if ($this->sessionConfigured()) {
        $this->storeCurrentUrl($request, $session);

        $this->addCookieToResponse($response, $session);
    }

    return $response;
}

public function terminate($request, $response)
{
    if ($this->sessionHandled && $this->sessionConfigured() && ! $this->usingCookieSessions()) {
        $this->manager->driver()->save();
    }
}
```

`session` 服务的中间件在 `http` 会话前与会话后都有处理。

在会话前，

- `laravel` 试图从 `cookies` 中获取 `sessionId`；
- 利用 `sessionId` 读取服务器中的 `session` 数据；
- 将 `session` 对象存入 `request` 中；
- `session` 垃圾回收

在会话后，

- 存储当前的 `url` 作为 `session` 的 `PreviousUrl`
- 将当前的 `session` 存入浏览器 `cookies` 中
- 保存当前的 `session` 数据到存储器驱动

### startSession

`startSession` 函数进行了 `session` 的启动工作：

```php
public function __construct(SessionManager $manager)
{
    $this->manager = $manager;
}
    
protected function startSession(Request $request)
{
    return tap($this->getSession($request), function ($session) use ($request) {
        $session->setRequestOnHandler($request);

        $session->start();
    });
}

public function getSession(Request $request)
{
    return tap($this->manager->driver(), function ($session) use ($request) {
        $session->setId($request->cookies->get($session->getName()));
    });
}
```

#### session 的门面类 sessionManager

代码很简洁，`session` 服务启动的逻辑被包含在了 `sessionManager` 中，`sessionManager` 是 `session` 服务的门面类，负责 `session` 服务的驱动加载与数据操作。

首先我们先看看 `SessionManager`:

```php
namespace Illuminate\Session;

use Illuminate\Support\Manager;

class SessionManager extends Manager 
{

}

```

`SessionManager` 继承 `Manager` 类：

```php
namespace Illuminate\Support;

abstract class Manager
{
    public function driver($driver = null)
    {
        $driver = $driver ?: $this->getDefaultDriver();

        if (! isset($this->drivers[$driver])) {
            $this->drivers[$driver] = $this->createDriver($driver);
        }

        return $this->drivers[$driver];
    }
}

```

当我们调用 `driver` 函数的时候，程序就开始为 `session` 服务加载驱动，例如对数据库或者 `redis` 驱动，进行 `连接` 操作。

```php
public function getDefaultDriver()
{
    return $this->app['config']['session.driver'];
}

protected function createDriver($driver)
{
    $method = 'create'.Str::studly($driver).'Driver';

    if (isset($this->customCreators[$driver])) {
        return $this->callCustomCreator($driver);
    } elseif (method_exists($this, $method)) {
        return $this->$method();
    }

    throw new InvalidArgumentException("Driver [$driver] not supported.");
}
```

#### session 驱动持久化类 SessionHandler


`FileSessionHandler` 这个类就是驱动，它继承 `SessionHandlerInterface` 基类，任何对 `session` 的读取、添加、删除、更新等等操作最后都要通过这个驱动类进行持久化。

-  `file` 驱动：

`file` 驱动的核心是 `Filesystem`，该类是 Ioc 容器创建的：

```php
protected function createFileDriver()
{
    return $this->createNativeDriver();
}

protected function createNativeDriver()
{
    $lifetime = $this->app['config']['session.lifetime'];

    return $this->buildSession(new FileSessionHandler(
        $this->app['files'], $this->app['config']['session.files'], $lifetime
    ));
}

namespace Illuminate\Session;

class FileSessionHandler implements SessionHandlerInterface
{
    public function __construct(Filesystem $files, $path, $minutes)
    {
        $this->path = $path;
        $this->files = $files;
        $this->minutes = $minutes;
    }
}

```

- redis 驱动

`redis` 驱动并不是直接创建 `redis`，而是利用了 `laravel` 的缓存 `cache` 系统创建 `redis` 驱动，然后对 `redis` 驱动进行连接操作： 

```php
protected function createRedisDriver()
{
    $handler = $this->createCacheHandler('redis');

    $handler->getCache()->getStore()->setConnection(
        $this->app['config']['session.connection']
    );

    return $this->buildSession($handler);
}


protected function createCacheHandler($driver)
{
    $store = $this->app['config']->get('session.store') ?: $driver;

    return new CacheBasedSessionHandler(
        clone $this->app['cache']->store($store),
        $this->app['config']['session.lifetime']
    );
}

class CacheBasedSessionHandler implements SessionHandlerInterface
{
    public function __construct(CacheContract $cache, $minutes)
    {
        $this->cache = $cache;
        $this->minutes = $minutes;
    }
}
```

#### session 数据操作类

`buildSession` 函数将会返回 `Store` 类，这个 `Store` 类实际上 `session` 服务数据操作的实质类，任何对 `session` 数据的操作实际上调用的都是 `Store` 类：

```php
protected function buildSession($handler)
{
    if ($this->app['config']['session.encrypt']) {
        return $this->buildEncryptedSession($handler);
    } else {
        return new Store($this->app['config']['session.cookie'], $handler);
    }
}

protected function buildEncryptedSession($handler)
{
    return new EncryptedStore(
        $this->app['config']['session.cookie'], $handler, $this->app['encrypter']
    );
}

public function __call($method, $parameters)
{
    return $this->driver()->$method(...$parameters);
}
```

如果需要对 `session` 进行加密，那么就会创建一个 `EncryptedStore` 类，该类继承 `Store` 类。

#### setId

`session` 驱动建立之后，就要进行 `sessionId` 的设置，如果 `cookie` 中存在 `sessionId`，我们就会从中获取，否则我们就需要重新生成新的 `sessionId`

```php
public function setId($id)
{
    $this->id = $this->isValidId($id) ? $id : $this->generateSessionId();
}

public function isValidId($id)
{
    return is_string($id) && ctype_alnum($id) && strlen($id) === 40;
}

protected function generateSessionId()
{
    return Str::random(40);
}

```

#### session--start

一切准备就绪后，我们就要启动 `session`，如果当前请求存在未过期 `session`，那么就要利用 `session` 驱动将数据读取出来：

```php
public function start()
{
    $this->loadSession();

    if (! $this->has('_token')) {
        $this->regenerateToken();
    }

    return $this->started = true;
}

protected function loadSession()
{
    $this->attributes = array_merge($this->attributes, $this->readFromHandler());
}
```
`readFromHandler` 函数就是读取 `session` 的过程：

```php
protected function readFromHandler()
{
    if ($data = $this->handler->read($this->getId())) {
        $data = @unserialize($this->prepareForUnserialize($data));

        if ($data !== false && ! is_null($data) && is_array($data)) {
            return $data;
        }
    }

    return [];
}
```

- 未加密 session 数据的加载

对于未加密的 `session` 来说，`prepareForUnserialize` 直接返回了数据：

```php
protected function prepareForUnserialize($data)
{
    return $data;
}
```

- 加密 session 数据

```php
protected function prepareForUnserialize($data)
{
    try {
        return $this->encrypter->decrypt($data);
    } catch (DecryptException $e) {
        return serialize([]);
    }
}
```

- file 驱动

```php
public function read($sessionId)
{
    if ($this->files->exists($path = $this->path.'/'.$sessionId)) {
        if (filemtime($path) >= Carbon::now()->subMinutes($this->minutes)->getTimestamp()) {
            return $this->files->get($path, true);
        }
    }

    return '';
}
```

- redis 驱动

```php
public function read($sessionId)
{
    return $this->cache->get($sessionId, '');
}
```

### session 垃圾回收

`session` 的垃圾回收用于随机性地删除旧 `session` 数据。由于某些驱动，例如 `FileSessionHandler`， 程序不会定期删除那些已经过时的 `session` 文件，那么 `session` 文件一定会越来越多，所以我们就需要一种垃圾回收机制：
 

```php
protected function collectGarbage(Session $session)
{
    $config = $this->manager->getSessionConfig();

    if ($this->configHitsLottery($config)) {
        $session->getHandler()->gc($this->getSessionLifetimeInSeconds());
    }
}

protected function configHitsLottery(array $config)
{
    return random_int(1, $config['lottery'][1]) <= $config['lottery'][0];
}
```

`configHitsLottery` 函数就是判断当前是否被随机要进行垃圾回收任务。这种随机性概率由 `lottery` 来设置。

`FileSessionHandler` 的垃圾回收：

```php
public function gc($lifetime)
{
    $files = Finder::create()
                ->in($this->path)
                ->files()
                ->ignoreDotFiles(true)
                ->date('<= now - '.$lifetime.' seconds');

    foreach ($files as $file) {
        $this->files->delete($file->getRealPath());
    }
}

```

### 存储前一页

很多时候我们都需要从 `session` 中获取前一页的地址，例如用户授权失败就会返回上一页等等情景。

```php
protected function storeCurrentUrl(Request $request, $session)
{
    if ($request->method() === 'GET' && $request->route() && ! $request->ajax()) {
        $session->setPreviousUrl($request->fullUrl());
    }
}

public function setPreviousUrl($url)
{
    $this->put('_previous.url', $url);
}

```

### 中间件的结束

当请求结束时，会调用中间件的 `terminate` 函数，这里程序会将新的 `session` 数据持久化到各个驱动器中：

```php
public function terminate($request, $response)
{
    if ($this->sessionHandled && $this->sessionConfigured() && ! $this->usingCookieSessions()) {
        $this->manager->driver()->save();
    }
}

```

`session` 的保存：

```php
public function save()
{
    $this->ageFlashData();

    $this->handler->write($this->getId(), $this->prepareForStorage(
        serialize($this->attributes)
    ));

    $this->started = false;
}
```

`session` 的保存会删除需要 `flash` 的闪存数据，也就是只想用于下一次请求的数据：

```php
public function ageFlashData()
{
    $this->forget($this->get('_flash.old', []));

    $this->put('_flash.old', $this->get('_flash.new', []));

    $this->put('_flash.new', []);
}
```

对于不加密的数据，保存前的 `prepareForStorage` 不会对数据进行任何操作：

```php
protected function prepareForStorage($data)
{
    return $data;
}
```

对于加密的数据，则需要事先加密：

```php
protected function prepareForStorage($data)
{
    return $this->encrypter->encrypt($data);
}
```

## session 数据操作

### get 函数

当我们想要获取 `session` 中的数据时，我们经常使用 `get` 方法 

```php
public function show(Request $request, $id)
{
    $value = $request->session()->get('key');

    //
}
```

`get` 方法首先会调用 `sessionManager` 的魔术方法：

```php
public function __call($method, $parameters)
{
    return $this->driver()->$method(...$parameters);
}
```

`driver` 函数会返回 `Store` 对象，调用 `get` 方法

```php
public function get($key, $default = null)
{
    return Arr::get($this->attributes, $key, $default);
}
```

我们从上一节知道，在 `startSession` 中间件启动后，`session` 数据已经加载到了 `store` 对象中，因此获取数据很简单：

```php
public function get($key, $default = null)
{
    return Arr::get($this->attributes, $key, $default);
}
```

### all 函数

`all` 函数可以取出所有的 `session` 数据

```php
public function all()
{
    return $this->attributes;
}
```

### has 函数

要确定 Session 中是否存在某个值，可以使用 `has` 方法。如果该值存在且不为 `null`，那么 `has` 方法会返回 `true`：

```php
public function has($key)
{
    return ! collect(is_array($key) ? $key : func_get_args())->contains(function ($key) {
        return is_null($this->get($key));
    });
}
```

### exists 函数

要确定 `Session` 中是否存在某个值，即使其值为 `null`，也可以使用 `exists` 方法。如果值存在，则 `exists` 方法返回 `true`

```php
public function exists($key)
{
    return ! collect(is_array($key) ? $key : func_get_args())->contains(function ($key) {
        return ! Arr::exists($this->attributes, $key);
    });
}
```

### put 方法

要存储数据到 `Session`，可以使用 `put` 方法

```php
public function put($key, $value = null)
{
    if (! is_array($key)) {
        $key = [$key => $value];
    }

    foreach ($key as $arrayKey => $arrayValue) {
        Arr::set($this->attributes, $arrayKey, $arrayValue);
    }
}
```

### push 方法

`push` 方法可以将一个新的值添加到 `Session` 数组内。

```php
public function push($key, $value)
{
    $array = $this->get($key, []);

    $array[] = $value;

    $this->put($key, $array);
}
```

### remember 方法

`remember` 方法用于有即取，无即存的情况：

```php
public function remember($key, Closure $callback)
{
    if (! is_null($value = $this->get($key))) {
        return $value;
    }

    return tap($callback(), function ($value) use ($key) {
        $this->put($key, $value);
    });
}

```

### increment 方法

`increment` 方法用于增加某 `session` 数据的值：

```php
public function increment($key, $amount = 1)
{
    $this->put($key, $value = $this->get($key, 0) + $amount);

    return $value;
}
```

### decrement 方法

```php
public function decrement($key, $amount = 1)
{
    return $this->increment($key, $amount * -1);
}

```

### pull 方法

`pull` 方法可以只用一条语句就从 `Session` 检索并且删除一个项目：

```php
public function pull($key, $default = null)
{
    return Arr::pull($this->attributes, $key, $default);
}
```

### flash 闪存数据

有时候你仅想在下一个请求之前在 `Session` 中存入数据，你可以使用 `flash` 方法。使用这个方法保存在 `session` 中的数据，只会保留到下个 `HTTP` 请求到来之前，然后就会被删除。闪存数据主要用于短期的状态消息

```php
public function flash($key, $value)
{
    $this->put($key, $value);

    $this->push('_flash.new', $key);

    $this->removeFromOldFlashData([$key]);
}

protected function removeFromOldFlashData(array $keys)
{
    $this->put('_flash.old', array_diff($this->get('_flash.old', []), $keys));
}
```

闪存数据的实现很简单，`session` 中会维护两个数组：`_flash.new`、`_flash.old` ,每次 `session` 结束前，都会删除 `_flash.old` 中的存储的 `key` 对应存储在 `session` 的 `value`。

### now 方法

`now` 方法用于存储只有本次请求采用的数据

```php
 public function now($key, $value)
{
    $this->put($key, $value);

    $this->push('_flash.old', $key);
}
```

### reflash 方法

如果需要保留闪存数据给更多请求，可以使用 reflash 方法，这将会将所有的闪存数据保留给其他请求。

```php
public function reflash()
{
    $this->mergeNewFlashes($this->get('_flash.old', []));

    $this->put('_flash.old', []);
}
```
这样，`_flash.old` 中的数据就会被合并到 `_flash.new` 中。


### keep 方法

只想保留特定的闪存数据给更多请求，则可以使用 keep 方法:

```php
public function keep($keys = null)
{
    $this->mergeNewFlashes($keys = is_array($keys) ? $keys : func_get_args());

    $this->removeFromOldFlashData($keys);
}
```

### forget 方法

forget 方法可以从 Session 内删除一条数据。

```php
public function forget($keys)
{
    Arr::forget($this->attributes, $keys);
}
```

### flush 方法

如果你想删除 Session 内所有数据，可以使用 flush 方法：

```ohp
public function flush()
{
    $this->attributes = [];
}
```

### 重新生成 Session ID

重新生成 `Session ID`，通常是为了防止恶意用户利用 `session fixation` 对应用进行攻击。如果使用了内置函数 `LoginController`，Laravel 会自动重新生成身份验证中 `Session ID`。否则，你需要手动使用 `regenerate` 方法重新生成 `Session ID`。

```php
public function regenerate($destroy = false)
{
    return $this->migrate($destroy);
}

public function migrate($destroy = false)
{
    if ($destroy) {
        $this->handler->destroy($this->getId());
    }

    $this->setExists(false);

    $this->setId($this->generateSessionId());

    return true;
}
```