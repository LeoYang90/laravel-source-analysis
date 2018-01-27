# Laravel Broadcast——广播系统源码剖析

## 前言

在现代的 `web` 应用程序中，`WebSockets` 被用来实现需要实时、即时更新的接口。当服务器上的数据被更新后，更新信息将通过 `WebSocket` 连接发送到客户端等待处理。相比于不停地轮询应用程序，`WebSocket` 是一种更加可靠和高效的选择。

我们先用一个电子商务网站作为例子来概览一下事件广播。当用户在查看自己的订单时，我们不希望他们必须通过刷新页面才能看到状态更新。我们希望一旦有更新时就主动将更新信息广播到客户端。

`laravel` 的广播系统和队列系统类似，需要两个进程协作，一个是 `laravel` 的 `web` 后台系统，另一个是 `Socket.IO` 服务器系统。具体的流程是页面加载时，网页 `js` 程序 `Laravel Echo` 与 `Socket.IO` 服务器建立连接， `laravel` 发起通过驱动发布广播，`Socket.IO` 服务器接受广播内容，对连接的客户端网页推送信息，以达到网页实时更新的目的。

`laravel` 发起广播的方式有两种，`redis` 与 `pusher`。对于 `redis` 来说，需要支持 `Socket.IO` 服务器系统，官方推荐 `nodejs` 为底层的 `tlaverdure/laravel-echo-server`。对于 `pusher` 来说，该第三方服务包含了驱动与 `Socket.IO` 服务器。

本文将会介绍 `redis` 为驱动的广播源码，由于 `laravel-echo-server` 是 `nodejs` 编写，本文也无法介绍 `Socket.IO` 方面的内容。

## 广播系统服务的启动

和其他服务类似，广播系统服务的注册实质上就是对 `Ioc` 容器注册门面类，广播系统的门面类是 `BroadcastManager`:

```php
class BroadcastServiceProvider extends ServiceProvider
{
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
}
```

除了注册 `BroadcastManager`，`BroadcastServiceProvider` 还进行了广播驱动的启动：

```php
public function connection($driver = null)
{
    return $this->driver($driver);
}

public function driver($name = null)
{
    $name = $name ?: $this->getDefaultDriver();

    return $this->drivers[$name] = $this->get($name);
}

protected function get($name)
{
    return isset($this->drivers[$name]) ? $this->drivers[$name] : $this->resolve($name);
}

protected function resolve($name)
{
    $config = $this->getConfig($name);

    if (is_null($config)) {
        throw new InvalidArgumentException("Broadcaster [{$name}] is not defined.");
    }

    if (isset($this->customCreators[$config['driver']])) {
        return $this->callCustomCreator($config);
    }

    $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

    if (! method_exists($this, $driverMethod)) {
        throw new InvalidArgumentException("Driver [{$config['driver']}] is not supported.");
    }

    return $this->{$driverMethod}($config);
}

protected function createRedisDriver(array $config)
{
    return new RedisBroadcaster(
        $this->app->make('redis'), Arr::get($config, 'connection')
    );
}
```

## 广播信息的发布

广播信息的发布与事件的发布大致相同，要告知 `Laravel` 一个给定的事件是广播类型，只需在事件类中实现 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 接口即可。该接口已经被导入到所有由框架生成的事件类中，所以可以很方便地将它添加到自己的事件中。

`ShouldBroadcast` 接口要求你实现一个方法：`broadcastOn`. `broadcastOn` 方法返回一个频道或一个频道数组，事件会被广播到这些频道。频道必须是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的实例。`Channel` 实例表示任何用户都可以订阅的公开频道，而 `PrivateChannels` 和 `PresenceChannels` 则表示需要 频道授权 的私有频道：

```php
class ServerCreated implements ShouldBroadcast
{
    use SerializesModels;

    public $user;
    
    //默认情况下，每一个广播事件都被添加到默认的队列上，默认的队列连接在 queue.php 配置文件中指定。可以通过在事件类中定义一个 broadcastQueue 属性来自定义广播器使用的队列。该属性用于指定广播使用的队列名称：
    public $broadcastQueue = 'your-queue-name';

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function broadcastOn()
    {
        return new PrivateChannel('user.'.$this->user->id);
    }
    
    //Laravel 默认会使用事件的类名作为广播名称来广播事件，自定义：
    public function broadcastAs()
	{
	    return 'server.created';
	}
	
	//想更细粒度地控制广播数据:
	public function broadcastWith()
	{
	    return ['id' => $this->user->id];
	}
	
	//有时，想在给定条件为 true ，才广播事件:
	public function broadcastWhen()
	{
	    return $this->value > 100;
	}
}
```

然后，只需要像平时那样触发事件。一旦事件被触发，一个队列任务会自动广播事件到你指定的广播驱动器上。

当一个事件被广播时，它所有的 `public` 属性会自动被序列化为广播数据，这允许你在你的 `JavaScript` 应用中访问事件的公有数据。因此，举个例子，如果你的事件有一个公有的 `$user` 属性，它包含了一个 `Elouqent` 模型，那么事件的广播数据会是：

```php
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}

```

## 广播发布的源码

广播的发布与事件的触发是一体的，具体的流程我们已经在 `event` 的源码中介绍清楚了，现在我们来看唯一的不同：

```php
public function dispatch($event, $payload = [], $halt = false)
    {
        list($event, $payload) = $this->parseEventAndPayload(
            $event, $payload
        );

        if ($this->shouldBroadcast($payload)) {
            $this->broadcastEvent($payload[0]);
        }

        ...
    }
    
    protected function shouldBroadcast(array $payload)
    {
        return isset($payload[0]) && $payload[0] instanceof ShouldBroadcast;
    }
    
    protected function broadcastEvent($event)
    {
        $this->container->make(BroadcastFactory::class)->queue($event);
    }
```

可见，关键之处在于 `BroadcastManager` 的 `quene` 方法：

```php
public function queue($event)
{
    $connection = $event instanceof ShouldBroadcastNow ? 'sync' : null;

    if (is_null($connection) && isset($event->connection)) {
        $connection = $event->connection;
    }

    $queue = null;

    if (isset($event->broadcastQueue)) {
        $queue = $event->broadcastQueue;
    } elseif (isset($event->queue)) {
        $queue = $event->queue;
    }

    $this->app->make('queue')->connection($connection)->pushOn(
        $queue, new BroadcastEvent(clone $event)
    );
}
```

可见，`quene` 方法将广播事件包装为事件类，并且通过队列发布，我们接下来看这个事件类的处理：

```php
class BroadcastEvent implements ShouldQueue
{
	public function handle(Broadcaster $broadcaster)
    {
        $name = method_exists($this->event, 'broadcastAs')
                ? $this->event->broadcastAs() : get_class($this->event);

        $broadcaster->broadcast(
            array_wrap($this->event->broadcastOn()), $name,
            $this->getPayloadFromEvent($this->event)
        );
    }

	protected function getPayloadFromEvent($event)
    {
        if (method_exists($event, 'broadcastWith')) {
            return array_merge(
                $event->broadcastWith(), ['socket' => data_get($event, 'socket')]
            );
        }

        $payload = [];

        foreach ((new ReflectionClass($event))->getProperties(ReflectionProperty::IS_PUBLIC) as $property) {
            $payload[$property->getName()] = $this->formatProperty($property->getValue($event));
        }

        return $payload;
    }
    
    protected function formatProperty($value)
    {
        if ($value instanceof Arrayable) {
            return $value->toArray();
        }

        return $value;
    }
}
```

可见该事件主要调用 `broadcaster` 的 `broadcast` 方法，我们这里讲 `redis` 的发布：

```php
class RedisBroadcaster extends Broadcaster
{
	public function broadcast(array $channels, $event, array $payload = [])
    {
        $connection = $this->redis->connection($this->connection);

        $payload = json_encode([
            'event' => $event,
            'data' => $payload,
            'socket' => Arr::pull($payload, 'socket'),
        ]);

        foreach ($this->formatChannels($channels) as $channel) {
            $connection->publish($channel, $payload);
        }
    }
}

protected function formatChannels(array $channels)
{
    return array_map(function ($channel) {
        return (string) $channel;
    }, $channels);
}
```

`broadcast` 方法运用了 `redis` 的 `publish` 方法，对 `redis` 进行了频道的信息发布。

## 频道授权

对于私有频道，用户只有被授权后才能监听。实现过程是用户向 `Laravel` 应用程序发起一个携带频道名称的 `HTTP` 请求，应用程序判断该用户是否能够监听该频道。在使用 `Laravel Echo` 时，上述 `HTTP` 请求会被自动发送；尽管如此，仍然需要定义适当的路由来响应这些请求。

### 定义授权路由

我们可以在 Laravel 里很容易地定义路由来响应频道授权请求。

```php
Broadcast::routes();

```
`Broadcast::routes` 方法会自动把它的路由放进 `web` 中间件组中；另外，如果你想对一些属性自定义，可以向该方法传递一个包含路由属性的数组

```php
Broadcast::routes($attributes);

```
### 定义授权回调

接下来，我们需要定义真正用于处理频道授权的逻辑。这是在 `routes/channels.php` 文件中完成。在该文件中，你可以用 `Broadcast::channel` 方法来注册频道授权回调函数：

```php
Broadcast::channel('order.{orderId}', function ($user, $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接收两个参数：频道名称和一个回调函数，该回调通过返回 `true` 或 `false` 来表示用户是否被授权监听该频道。

所有的授权回调接收当前被认证的用户作为第一个参数，任何额外的通配符参数作为后续参数。在本例中，我们使用 `{orderId}` 占位符来表示频道名称的「ID」部分是通配符。

### 授权回调模型绑定

就像 `HTTP` 路由一样，频道路由也可以利用显式或隐式 路由模型绑定。例如，相比于接收一个字符串或数字类型的 `order ID`，你也可以请求一个真正的 `Order` 模型实例:

```php
Broadcast::channel('order.{order}', function ($user, Order $order) {
    return $user->id === $order->user_id;
});

```

## 频道授权源码分析

### 授权路由

```php
class BroadcastManager implements FactoryContract
{
	public function routes(array $attributes = null)
    {
        if ($this->app->routesAreCached()) {
            return;
        }

        $attributes = $attributes ?: ['middleware' => ['web']];

        $this->app['router']->group($attributes, function ($router) {
            $router->post('/broadcasting/auth', BroadcastController::class.'@authenticate');
        });
    }
}
```
频道专门有 `Controller` 来处理授权服务：

```php
class BroadcastController extends Controller
{
    public function authenticate(Request $request)
    {
        return Broadcast::auth($request);
    }
}

```
当 `Socket Io` 服务器对 `javascript` 程序推送数据的时候，首先会经过该 `controller` 进行授权验证：

```php
public function auth($request)
{
    if (Str::startsWith($request->channel_name, ['private-', 'presence-']) &&
        ! $request->user()) {
        throw new HttpException(403);
    }

    $channelName = Str::startsWith($request->channel_name, 'private-')
                        ? Str::replaceFirst('private-', '', $request->channel_name)
                        : Str::replaceFirst('presence-', '', $request->channel_name);

    return parent::verifyUserCanAccessChannel(
        $request, $channelName
    );
}
```

`verifyUserCanAccessChannel` 根据频道与其绑定的闭包函数来验证该频道是否可以通过授权：

```php
protected function verifyUserCanAccessChannel($request, $channel)
{
    foreach ($this->channels as $pattern => $callback) {
        if (! Str::is(preg_replace('/\{(.*?)\}/', '*', $pattern), $channel)) {
            continue;
        }

        $parameters = $this->extractAuthParameters($pattern, $channel, $callback);

        if ($result = $callback($request->user(), ...$parameters)) {
            return $this->validAuthenticationResponse($request, $result);
        }
    }

    throw new HttpException(403);
}
```

由于频道的命名经常带有 `userid` 等参数，因此判断频道之前首先要把 `channels` 中的频道名转为通配符 `*`，例如 `order.{userid}` 转为 order.*，之后进行正则匹配。

`extractAuthParameters` 用于提取频道的闭包函数的参数，合并 `$request->user()` 之后调用闭包函数。

```php
protected function extractAuthParameters($pattern, $channel, $callback)
{
    $callbackParameters = (new ReflectionFunction($callback))->getParameters();

    return collect($this->extractChannelKeys($pattern, $channel))->reject(function ($value, $key) {
        return is_numeric($key);
    })->map(function ($value, $key) use ($callbackParameters) {
        return $this->resolveBinding($key, $value, $callbackParameters);
    })->values()->all();
}

protected function extractChannelKeys($pattern, $channel)
{
    preg_match('/^'.preg_replace('/\{(.*?)\}/', '(?<$1>[^\.]+)', $pattern).'/', $channel, $keys);

    return $keys;
}

public function validAuthenticationResponse($request, $result)
{
    if (is_bool($result)) {
        return json_encode($result);
    }

    return json_encode(['channel_data' => [
        'user_id' => $request->user()->getKey(),
        'user_info' => $result,
    ]]);
}
```

`extractChannelKeys` 用于将 `order.{userid}` 与 `order.23` 中 `userid` 和 `23` 建立 `key`、`value` 关联。如果 `userid` 是 `User` 的主键，`resolveBinding` 还可以为其自动进行路由模型绑定。 