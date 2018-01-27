# Laravel Event——事件系统的启动与运行源码分析

## 前言

`Laravel` 的事件系统是一个简单的观察者模式，主要目的是用于代码的解耦，可以防止不同功能的代码耦合在一起。`laravel` 中事件系统由两部分构成，一个是事件的名称，事件的名称可以是个字符串，例如 `event.email`，也可以是一个事件类，例如 `App\Events\OrderShipped`；另一个是事件的 `listener`，可以是一个闭包，还可以是监听类，例如 `App\Listeners\SendShipmentNotification`。

## 事件服务的注册

事件服务的注册分为两部分，一个是 `Application` 启动时所调用的 `registerBaseServiceProviders` 函数：

```php
protected function registerBaseServiceProviders()
{
    $this->register(new EventServiceProvider($this));

    $this->register(new LogServiceProvider($this));

    $this->register(new RoutingServiceProvider($this));
}
```

其中的 `EventServiceProvider` 是 `/Illuminate/Events/EventServiceProvider`:

```php
public function register()
{
    $this->app->singleton('events', function ($app) {
        return (new Dispatcher($app))->setQueueResolver(function () use ($app) {
            return $app->make(QueueFactoryContract::class);
        });
    });
}

```
这部分为 `Ioc` 容器注册了 `events` 实例，`Dispatcher` 就是 `events` 真正的实现类。`QueueResolver` 是队列化事件的实现。

另一个注册是普通注册类 `/app/Providers/EventServiceProvider` :

```php
class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        'App\Events\SomeEvent' => [
            'App\Listeners\EventListener',
        ],
    ];

    public function boot()
    {
        parent::boot();
        //
    }
}

```

这个注册类的主要作用是事件系统的启动，这个类继承自 `/Illuminate/Foundation/Support/Providers/EventServiceProvider`:

```php
class EventServiceProvider extends ServiceProvider
{
    protected $listen = [];

    protected $subscribe = [];

    public function boot()
    {
        foreach ($this->listens() as $event => $listeners) {
            foreach ($listeners as $listener) {
                Event::listen($event, $listener);
            }
        }

        foreach ($this->subscribe as $subscriber) {
            Event::subscribe($subscriber);
        }
    }
}

```
可以看到，事件系统的启动主要是进行事件系统的监听与订阅。

## 事件系统的监听 listen

所谓的事件监听，就是将事件名与闭包函数，或者事件类与监听类之间建立关联。

```php
public function listen($events, $listener)
{
    foreach ((array) $events as $event) {
        if (Str::contains($event, '*')) {
            $this->setupWildcardListen($event, $listener);
        } else {
            $this->listeners[$event][] = $this->makeListener($listener);
        }
    }
}

protected function setupWildcardListen($event, $listener)
{
    $this->wildcards[$event][] = $this->makeListener($listener, true);
}
```
对于有通配符的事件名，会统一放入 `wildcards` 数组中，`makeListener` 是创建事件的关键:

```php
public function makeListener($listener, $wildcard = false)
{
    if (is_string($listener)) {
        return $this->createClassListener($listener, $wildcard);
    }

    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return $listener($event, $payload);
        } else {
            return $listener(...array_values($payload));
        }
    };
}
```
创建监听者的时候，会判断监听对象是监听类还是闭包函数。

对于闭包监听来说，`makeListener` 会再包上一层闭包函数，根据是否含有通配符来确定具体的参数。

对于监听类来说，会继续 `createClassListener`:

```php
public function createClassListener($listener, $wildcard = false)
{
    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return call_user_func($this->createClassCallable($listener), $event, $payload);
        } else {
            return call_user_func_array(
                $this->createClassCallable($listener), $payload
            );
        }
    };
}

protected function createClassCallable($listener)
{
    list($class, $method) = $this->parseClassCallable($listener);

    if ($this->handlerShouldBeQueued($class)) {
        return $this->createQueuedHandlerCallable($class, $method);
    } else {
        return [$this->container->make($class), $method];
    }
}
```

对于监听类来说，程序首先会判断监听类对应的函数：

```php
protected function parseClassCallable($listener)
{
    return Str::parseCallback($listener, 'handle');
}
```

如果未指定监听类的对应函数，那么会默认 `handle` 函数。

如果当前监听类是队列的话，会将任务推送给队列。

## 触发事件

事件的触发可以利用事件名，或者事件类的实例：

```php
public function dispatch($event, $payload = [], $halt = false)
{
    list($event, $payload) = $this->parseEventAndPayload(
        $event, $payload
    );

    if ($this->shouldBroadcast($payload)) {
        $this->broadcastEvent($payload[0]);
    }

    $responses = [];

    foreach ($this->getListeners($event) as $listener) {
        $response = $listener($event, $payload);

        if (! is_null($response) && $halt) {
            return $response;
        }

        if ($response === false) {
            break;
        }

        $responses[] = $response;
    }

    return $halt ? null : $responses;
}

```
`parseEventAndPayload` 函数利用传入参数是事件名还是事件类实例来确定监听类函数的参数：

```php
protected function parseEventAndPayload($event, $payload)
{
    if (is_object($event)) {
        list($payload, $event) = [[$event], get_class($event)];
    }

    return [$event, array_wrap($payload)];
}
```
如果是事件类的实例，那么监听函数的参数就是事件类自身；如果是事件类名，那么监听函数的参数就是触发事件时传入的参数。

获得事件与参数后，就要获取监听类：

```php
public function getListeners($eventName)
{
    $listeners = isset($this->listeners[$eventName]) ? $this->listeners[$eventName] : [];

    $listeners = array_merge(
        $listeners, $this->getWildcardListeners($eventName)
    );

    return class_exists($eventName, false)
                ? $this->addInterfaceListeners($eventName, $listeners)
                : $listeners;
}

```

寻找监听类的时候，也要从通配符监听器中寻找：

```php
protected function getWildcardListeners($eventName)
{
    $wildcards = [];

    foreach ($this->wildcards as $key => $listeners) {
        if (Str::is($key, $eventName)) {
            $wildcards = array_merge($wildcards, $listeners);
        }
    }

    return $wildcards;
}
```

如果监听类继承自其他类，那么父类也会一并当做监听类返回。

获得了监听类之后，就要调用监听类相应的函数。

触发事件时有一个参数 `halt`，这个参数如果是 `true` 的时候，只要有一个监听类返回了结果，那么就会立刻返回。例如：

```php
public function testHaltingEventExecution()
{
    unset($_SERVER['__event.test']);
    $d = new Dispatcher;
    $d->listen('foo', function ($foo) {
        $this->assertTrue(true);

        return 'here';
    });
    $d->listen('foo', function ($foo) {
        throw new Exception('should not be called');
    });
    $d->until('foo', ['bar']);
}

```

多个监听类在运行的时候，只要有一个返回了 `false`，那么就会中断事件。

### push 函数

`push` 函数可以将触发事件的参数事先设置好，这样触发的时候只要写入事件名即可，例如：

```php
public function testQueuedEventsAreFired()
{
    unset($_SERVER['__event.test']);
    $d = new Dispatcher;
    $d->push('update', ['name' => 'taylor']);
    $d->listen('update', function ($name) {
        $_SERVER['__event.test'] = $name;
    });

    $this->assertFalse(isset($_SERVER['__event.test']));
    $d->flush('update');
    $this->assertEquals('taylor', $_SERVER['__event.test']);
}
```

原理也很简单：

```php
public function push($event, $payload = [])
{
    $this->listen($event.'_pushed', function () use ($event, $payload) {
        $this->dispatch($event, $payload);
    });
}

public function flush($event)
{
    $this->dispatch($event.'_pushed');
}
```

## 数据库 Eloquent 的事件

数据库模型的事件的注册除了以上的方法还有另外两种，具体详情可以看：[Laravel 模型事件实现原理](https://laravel-china.org/articles/5465/event-realization-principle-of-laravel-model) ;

### 事件注册

- 静态方法定义

```php
class EventServiceProvider extends ServiceProvider
{
	public function boot()
	{
	    parent::boot();
	
	    User::saved(function(User$user) {
	
	    });
	
	    User::saved('UserSavedListener@saved');
	}
}

```

- 观察者

```php
class UserObserver
{
    public function created(User $user)
    {
        //
    }

    public function saved(User $user)
    {
        //
    }
}

```

然后在某个服务提供者的boot方法中注册观察者：

```php
class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        User::observe(UserObserver::class);
    }

    public function register()
    {
        //
    }
}

```

这两种方法都是向事件系统注册事件名 `eloquent.{$event}: {static::class}`:

- 静态方法

```php
public static function saved($callback)
{
    static::registerModelEvent('saved', $callback);
}

protected static function registerModelEvent($event, $callback)
{
    if (isset(static::$dispatcher)) {
        $name = static::class;

        static::$dispatcher->listen("eloquent.{$event}: {$name}", $callback);
    }
}
```

- 观察者

```php
public static function observe($class)
{
    $instance = new static;

    $className = is_string($class) ? $class : get_class($class);

    foreach ($instance->getObservableEvents() as $event) {
        if (method_exists($class, $event)) {
            static::registerModelEvent($event, $className.'@'.$event);
        }
    }
}

public function getObservableEvents()
{
    return array_merge(
        [
            'creating', 'created', 'updating', 'updated',
            'deleting', 'deleted', 'saving', 'saved',
            'restoring', 'restored',
        ],
        $this->observables
    );
}
```

### 事件触发

模型事件的触发需要调用  `fireModelEvent` 函数：

```php
protected function fireModelEvent($event, $halt = true)
{
    if (! isset(static::$dispatcher)) {
        return true;
    }

    $method = $halt ? 'until' : 'fire';

    $result = $this->fireCustomModelEvent($event, $method);

    return ! is_null($result) ? $result : static::$dispatcher->{$method}(
        "eloquent.{$event}: ".static::class, $this
    );
}

```

`fireCustomModelEvent` 是我们本文中着重讲的事件类与监听类的触发：

```php
protected function fireCustomModelEvent($event, $method)
{
    if (! isset($this->events[$event])) {
        return;
    }

    $result = static::$dispatcher->$method(new $this->events[$event]($this));

    if (! is_null($result)) {
        return $result;
    }
}
```

如果没有对应的事件后，会继续利用事件名进行触发。

`until` 是我们上一节讲的如果任意事件返回正确结果，就会直接返回，不会继续进行下一个事件。