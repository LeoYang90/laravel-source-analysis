# 前言

服务提供者是 `laravel` 框架的重要组成部分，承载着各种服务，自定义的应用以及所有 `Laravel` 的核心服务都是通过服务提供者启动。本文将会介绍服务提供者的源码分析，关于服务提供者的使用，请参考官方文档 ：[服务提供者](http://laravelacademy.org/post/6703.html)。

# 服务提供者的注册

服务提供者的启动由类 `\Illuminate\Foundation\Bootstrap\RegisterProviders::class` 负责，该类用于加载所有服务提供者的 `register` 函数，并保存延迟加载的服务的信息，以便实现延迟加载。

```php
class RegisterProviders
{
    public function bootstrap(Application $app)
    {
        $app->registerConfiguredProviders();
    }
}

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
	public function registerConfiguredProviders()
    {
        (new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath()))
                    ->load($this->config['app.providers']);
    }
    
    public function getCachedServicesPath()
    {
        return $this->bootstrapPath().'/cache/services.php';
    }
}
```
以上可以看出，所有服务提供者都在配置文件 `app.php` 文件的 `providers` 数组中。类 `ProviderRepository` 负责所有的服务加载功能：

```php
class ProviderRepository
{
	public function load(array $providers)
    {
        $manifest = $this->loadManifest();

        if ($this->shouldRecompile($manifest, $providers)) {
            $manifest = $this->compileManifest($providers);
        }

        foreach ($manifest['when'] as $provider => $events) {
            $this->registerLoadEvents($provider, $events);
        }

        foreach ($manifest['eager'] as $provider) {
            $this->app->register($provider);
        }

        $this->app->addDeferredServices($manifest['deferred']);
    }
}
```

## 加载服务缓存文件

`laravel` 会把所有的服务整理起来，作为缓存写在缓存文件中：

```php
return array (
	'providers' => 
	array (
	  0 => 'Illuminate\\Auth\\AuthServiceProvider',
	  1 => 'Illuminate\\Broadcasting\\BroadcastServiceProvider',
	  ...
	),
	
	'eager' => 
	array (
      0 => 'Illuminate\\Auth\\AuthServiceProvider',
      1 => 'Illuminate\\Cookie\\CookieServiceProvider',
      ...
    ),
    
    'deferred' => 
    array (
      'Illuminate\\Broadcasting\\BroadcastManager' => 'Illuminate\\Broadcasting\\BroadcastServiceProvider',
      'Illuminate\\Contracts\\Broadcasting\\Factory' => 'Illuminate\\Broadcasting\\BroadcastServiceProvider',
      ...
    ),
    
    'when' => 
    array (
      'Illuminate\\Broadcasting\\BroadcastServiceProvider' => 
      array (
      ),
      ...
    ),
```

- 缓存文件中 `providers` 放入了所有自定义和框架核心的服务。
- `eager` 数组中放入了所有需要立即启动的服务提供者。
- `deferred` 数组中放入了所有需要延迟加载的服务提供者。
- `when` 放入了延迟加载需要激活的事件。

加载服务提供者缓存文件：

```php
public function loadManifest()
{
    if ($this->files->exists($this->manifestPath)) {
        $manifest = $this->files->getRequire($this->manifestPath);

        if ($manifest) {
            return array_merge(['when' => []], $manifest);
        }
    }
}
```

## 编译服务提供者

若 `laravel` 中的服务提供者没有缓存文件或者有变动，那么就会重新生成缓存文件：

```php
public function shouldRecompile($manifest, $providers)
{
    return is_null($manifest) || $manifest['providers'] != $providers;
}

protected function compileManifest($providers)
{
    $manifest = $this->freshManifest($providers);

    foreach ($providers as $provider) {
        $instance = $this->createProvider($provider);

        if ($instance->isDeferred()) {
            foreach ($instance->provides() as $service) {
                $manifest['deferred'][$service] = $provider;
            }

            $manifest['when'][$provider] = $instance->when();
        }

        else {
            $manifest['eager'][] = $provider;
        }
    }

    return $this->writeManifest($manifest);
}

protected function freshManifest(array $providers)
{
    return ['providers' => $providers, 'eager' => [], 'deferred' => []];
}
```
- 如果服务提供者是需要立即注册的，那么将会放入缓存文件中 `eager` 数组中。
- 如果服务提供者是延迟加载的，那么其函数 `provides()` 通常会提供服务别名，这个服务别名通常是向服务容器中注册的别名，别名将会放入缓存文件的 `deferred` 数组中。
- 延迟加载若有 `event` 事件激活，那么可以在 `when` 函数中写入事件类，并写入缓存文件的 `when` 数组中。

## 延迟服务提供者事件注册

延迟服务提供者除了利用 `IOC` 容器解析服务方式激活，还可以利用 `Event` 事件来激活：

```php
protected function registerLoadEvents($provider, array $events)
{
    if (count($events) < 1) {
        return;
    }

    $this->app->make('events')->listen($events, function () use ($provider) {
        $this->app->register($provider);
    });
}
```

## 注册即时启动的服务提供者

服务提供者的注册函数 `register()` 由类 `Application` 来调用：

```php
class Application extends Container implements ApplicationContract, HttpKernelInterface
{
	public function register($provider, $options = [], $force = false)
    {
        if (($registered = $this->getProvider($provider)) && ! $force) {
            return $registered;
        }

        if (is_string($provider)) {
            $provider = $this->resolveProvider($provider);
        }

        if (method_exists($provider, 'register')) {
            $provider->register();
        }

        $this->markAsRegistered($provider);

        if ($this->booted) {
            $this->bootProvider($provider);
        }

        return $provider;
    }
    
    public function getProvider($provider)
    {
        $name = is_string($provider) ? $provider : get_class($provider);

        return Arr::first($this->serviceProviders, function ($value) use ($name) {
            return $value instanceof $name;
        });
    }
    
    public function resolveProvider($provider)
    {
        return new $provider($this);
    }
    
    protected function markAsRegistered($provider)
    {
        $this->serviceProviders[] = $provider;

        $this->loadedProviders[get_class($provider)] = true;
    }
    
    protected function bootProvider(ServiceProvider $provider)
    {
        if (method_exists($provider, 'boot')) {
            return $this->call([$provider, 'boot']);
        }
    }
}
```

可以看出，服务提供者的注册过程：

- 判断当前服务提供者是否被注册过，如注册过直接返回对象
- 解析服务提供者
- 调用服务提供者的 `register` 函数
- 标记当前服务提供者已经注册完毕
- 若框架已经加载注册完毕所有的服务容器，那么就启动服务提供者的 `boot` 函数，该函数由于是 `call` 调用，所以支持依赖注入。

## 延迟服务提供者激活与注册

延迟服务提供者首先需要添加到 `Application` 中：

```php
public function addDeferredServices(array $services)
{
    $this->deferredServices = array_merge($this->deferredServices, $services);
}
```

我们之前说过，延迟服务提供者的激活注册有两种方法：事件与服务解析。

当特定的事件被激发后，就会调用 `Application` 的 `register` 函数，进而调用服务提供者的 `register` 函数，实现服务的注册。

当利用 `Ioc` 容器解析服务名时，例如解析服务名 `BroadcastingFactory`：

```php
class BroadcastServiceProvider extends ServiceProvider
{
	protected $defer = true;
	
	public function provides()
    {
        return [
            BroadcastManager::class,
            BroadcastingFactory::class,
            BroadcasterContract::class,
        ];
    }
}
```

```php
public function make($abstract)
{
    $abstract = $this->getAlias($abstract);

    if (isset($this->deferredServices[$abstract])) {
        $this->loadDeferredProvider($abstract);
    }

    return parent::make($abstract);
}

public function loadDeferredProvider($service)
{
    if (! isset($this->deferredServices[$service])) {
        return;
    }

    $provider = $this->deferredServices[$service];

    if (! isset($this->loadedProviders[$provider])) {
        $this->registerDeferredProvider($provider, $service);
    }
}
```

由 `deferredServices` 数组可以得知，`BroadcastingFactory` 为延迟服务，接着程序会利用函数 `loadDeferredProvider` 来加载延迟服务提供者，调用服务提供者的 `register` 函数，若当前的框架还未注册完全部服务。那么将会放入服务启动的回调函数中，以待服务启动时调用：

```php
public function registerDeferredProvider($provider, $service = null)
{
    if ($service) {
        unset($this->deferredServices[$service]);
    }

    $this->register($instance = new $provider($this));

    if (! $this->booted) {
        $this->booting(function () use ($instance) {
            $this->bootProvider($instance);
        });
    }
}
```

关于服务提供者的注册函数：

```php
class BroadcastServiceProvider extends ServiceProvider
{
    protected $defer = true;

    public function register()
    {
        $this->app->singleton(BroadcastManager::class, function ($app) {
            return new BroadcastManager($app);
        });

        $this->app->singleton(BroadcasterContract::class, function ($app) {
            return $app->make(BroadcastManager::class)->connection();
        });

        $this->app->alias(
            BroadcastManager::class, BroadcastingFactory::class
        );
    }

    public function provides()
    {
        return [
            BroadcastManager::class,
            BroadcastingFactory::class,
            BroadcasterContract::class,
        ];
    }
}

```

函数 `register` 为类 `BroadcastingFactory` 向 `Ioc` 容器绑定了特定的实现类 `BroadcastManager`，这样 `Ioc` 容器中的 `make` 函数：

```php
public function make($abstract)
{
    $abstract = $this->getAlias($abstract);

    if (isset($this->deferredServices[$abstract])) {
        $this->loadDeferredProvider($abstract);
    }

    return parent::make($abstract);
}
```

`parent::make($abstract)` 就会正确的解析服务 `BroadcastingFactory`。

因此函数 `provides()` 返回的元素一定都是 `register()` 向 `IOC` 容器中绑定的类名或者别名。这样当我们利用服务容器来利用 `App::make()` 解析这些类名的时候，服务容器才会根据服务提供者的 `register()` 函数中绑定的实现类，从而正确解析服务功能。

# 服务容器的启动

服务容器的启动由类 `\Illuminate\Foundation\Bootstrap\BootProviders::class` 负责：

```php
class BootProviders
{
    public function bootstrap(Application $app)
    {
        $app->boot();
    }
}

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
	public function boot()
    {
        if ($this->booted) {
            return;
        }

        $this->fireAppCallbacks($this->bootingCallbacks);

        array_walk($this->serviceProviders, function ($p) {
            $this->bootProvider($p);
        });

        $this->booted = true;

        $this->fireAppCallbacks($this->bootedCallbacks);
    }
    
    protected function bootProvider(ServiceProvider $provider)
    {
        if (method_exists($provider, 'boot')) {
            return $this->call([$provider, 'boot']);
        }
    }
}
```

