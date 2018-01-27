# Laravel Queue——消息队列任务与分发源码剖析

## 前言

在实际的项目开发中，我们经常会遇到需要轻量级队列的情形，例如发短信、发邮件等，这些任务不足以使用 `kafka`、`RabbitMQ` 等重量级的消息队列，但是又的确需要异步、重试、并发控制等功能。通常来说，我们经常会使用 `Redis`、`Beanstalk`、`Amazon SQS` 来实现相关功能，`laravel` 为此对不同的后台队列服务提供统一的 `API`，本文将会介绍应用最为广泛的 `redis` 队列。

本文参考文档资料：

[使用 Laravel Queue 不得不明白的知识](https://laravel-china.org/articles/3729/use-laravel-queue-to-understand-the-knowledge)

[Laravel 的消息队列剖析](https://laravel-china.org/articles/4169/analysis-of-laravel-message-queue)

## 背景知识

在讲解 `laravel` 的队列服务之前，我们要先说说基于 `redis` 的队列服务。首先，redis设计用来做缓存的，但是由于它自身的某种特性使得它可以用来做消息队列，

### redis 队列的数据结构

- List 链表

`redis` 做消息队列的特性例如FIFO（先入先出）很容易实现，只需要一个 `list` 对象从头取数据，从尾部塞数据即可。

相关的命令：（1）左侧入右侧出：lpush/rpop；（2）右侧入左侧出：rpush/lpop。

这个简单的消息队列很容易实现。

- Zset 有序集合

有些任务场景，并不需要任务立刻执行，而是需要延迟执行；有些任务很重要，需要在任务失败的时候重新尝试。这些功能仅仅依靠 `list` 是无法完成的。这个时候，就需要 `redis` 的有序集合。

`Redis` 有序集合和 `Redis` 集合类似，是不包含相同字符串的合集。它们的差别是，每个有序集合的成员都关联着一个评分 `score`，这个评分用于把有序集合中的成员按最低分到最高分排列。

单看有序集合和延迟任务并无关系，但是可以将有序集合的评分 `score` 设置为延时任务开启的时间，之后轮询这个有序集合，将到期的任务拿出来进行处理，这样就实现了延迟任务的功能。

对于重要的需要重试的任务，在任务执行之前，会将该任务放入有序集合中，设置任务最长的执行时间。若任务顺利执行完毕，该任务会在有序集合中删除。如果任务没有在规定时间内完成，那么该有序集合的任务将会被重新放入队列中。

相关命令：

(1) `ZADD` 添加一个或多个成员到有序集合，或者如果它已经存在更新其分数。

(2) `ZRANGEBYSCORE` 按分数返回一个成员范围的有序集合。

(3) `ZREMRANGEBYRANK` 在给定的索引之内删除所有成员的有序集合。

### laravel 队列服务的任务调度

队列服务的任务调度过程如下：

![](http://owql68l6p.bkt.clouddn.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

`laravel` 的队列服务由两个进程控制，一个是生产者，一个是消费者。这两个进程操纵了 `redis` 三个队列，其中一个 `List`，负责即时任务，两个 `Zset`，负责延时任务与待处理任务。

生产者负责向 `redis` 推送任务，如果是即时任务，默认就会向 `queue:default` 推送；如果是延时任务，就会向 `queue:default:delayed` 推送。

消费者轮询两个队列，不断的从队列中取出任务，先把任务放入 `queue:default:reserved` 中，再执行相关任务。如果任务执行成功，就会删除 `queue:default:reserved` 中的任务，否则会被重新放入 `queue:default:delayed` 队列中。

## laravel 队列服务的总体流程

任务分发流程：

![](http://owql68l6p.bkt.clouddn.com/QQ20171208-213853@2x.png)

任务处理器运作：

![](http://owql68l6p.bkt.clouddn.com/QQ20171208-213913@2x.png)


## laravel 队列服务的注册与启动

`laravel` 队列服务需要注册的服务比较多：

```php
class QueueServiceProvider extends ServiceProvider
{
	public function register()
    {
        $this->registerManager();

        $this->registerConnection();

        $this->registerWorker();

        $this->registerListener();

        $this->registerFailedJobServices();
    }
}

```

### registerManager 注册门面

`registerManager` 负责注册队列服务的门面类：

```php
protected function registerManager()
{
    $this->app->singleton('queue', function ($app) {
        return tap(new QueueManager($app), function ($manager) {
            $this->registerConnectors($manager);
        });
    });
}

public function registerConnectors($manager)
{
    foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
        $this->{"register{$connector}Connector"}($manager);
    }
}

protected function registerRedisConnector($manager)
{
    $manager->addConnector('redis', function () {
        return new RedisConnector($this->app['redis']);
    });
}
```

`QueueManager` 是队列服务的总门面，提供一切与队列相关的操作接口。`QueueManager` 中有一个成员变量 `$connectors`，该成员变量中存储着所有 `laravel` 支持的底层队列服务：'Database', 'Redis', 'Beanstalkd', 'Sqs'。

```php
class QueueManager implements FactoryContract, MonitorContract
{
	public function addConnector($driver, Closure $resolver)
    {
        $this->connectors[$driver] = $resolver;
    }
}

```

成员变量 `$connectors` 会被存储各种驱动的 `connector`，例如 `RedisConnector`、`SqsConnector`、`DatabaseConnector`、`BeanstalkdConnector`。


### registerConnection 底层队列连接服务

接下来，就要连接实现队列的底层服务了，例如 `redis`：

```php
protected function registerConnection()
{
    $this->app->singleton('queue.connection', function ($app) {
        return $app['queue']->connection();
    });
}

public function connection($name = null)
{
    $name = $name ?: $this->getDefaultDriver();

    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->resolve($name);

        $this->connections[$name]->setContainer($this->app);
    }

    return $this->connections[$name];
}

public function getDefaultDriver()
{
    return $this->app['config']['queue.default'];
}
```

`connection` 函数首先会获取 `连接` 名，没有 `连接` 名就会从 `config` 中获取默认的连接。

```php
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver'])
                    ->connect($config)
                    ->setConnectionName($name);
}

```
`resolve` 函数利用相应的底层驱动 `connector` 进行连接操作，也就是 `connect` 函数，该函数会返回 `RedisQueue`：

```php
class RedisConnector implements ConnectorInterface
{
	public function connect(array $config)
    {
        return new RedisQueue(
            $this->redis, $config['queue'],
            Arr::get($config, 'connection', $this->connection),
            Arr::get($config, 'retry_after', 60)
        );
    }
}

```

### registerWorker 消费者服务注册

消费者的注册服务会返回 `Illuminate\Queue\Worker` 类：

```php
protected function registerWorker()
{
    $this->app->singleton('queue.worker', function () {
        return new Worker(
            $this->app['queue'], $this->app['events'], $this->app[ExceptionHandler::class]
        );
    });
}

```

## laravel Bus 服务注册与启动

定义好自己想要的队列类之后，还需要将队列任务推送给底层驱动后台，例如 `redis`，一般会使用 `dispatch` 函数：

```php
Job::dispatch();

```
或者

```php
$job = (new ProcessPodcast($pocast));

dispatch($job);

```
`dispatch` 函数就是 `Bus` 服务，专门用于分发队列任务。

```php
class BusServiceProvider extends ServiceProvider
{
	public function register()
    {
        $this->app->singleton(Dispatcher::class, function ($app) {
            return new Dispatcher($app, function ($connection = null) use ($app) {
                return $app[QueueFactoryContract::class]->connection($connection);
            });
        });

        $this->app->alias(
            Dispatcher::class, DispatcherContract::class
        );

        $this->app->alias(
            Dispatcher::class, QueueingDispatcherContract::class
        );
    }
}
```

## 创建任务

### queue 设置

```php
'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => 'default',
        'retry_after' => 90,
    ],

```

一般来说，默认的 `redis` 配置如上，`connection` 是 `database` 中 `redis` 的连接名称；`queue` 是 `redis` 中的队列名称，值得注意的是，如果使用的是 `redis` 集群的话，这个需要使用 `key hash tag`，也就是 `{default}`;当任务运行超过 `retry_after` 这个时间后，该任务会被重新放入队列当中。

### 任务类的创建

- 任务类的结构很简单，一般来说只会包含一个让队列用来调用此任务的 `handle` 方法。

- 如果想要使得任务被推送到队列中，而不是同步执行，那么需要实现 `Illuminate\Contracts\Queue\ShouldQueue` 接口。

- 如果想要让任务推送到特定的连接中，例如 `redis` 或者 `sqs`，那么需要设置 `conneciton` 变量。

- 如果想要让任务推送到特定的队列中去，可以设置 `queue` 变量。

- 如果想要让任务延迟推送，那么需要设置 `delay` 变量。

- 如果想要设置任务至多重试的次数，可以使用 `tries` 变量；

- 如果想要设置任务可以运行的最大秒数，那么可以使用 `timeout` 参数。

- 如果想要手动访问队列，可以使用 `trait` : `Illuminate\Queue\InteractsWithQueue`。

- 如果队列监听器任务执行次数超过在工作队列中定义的最大尝试次数，监听器的 `failed` 方法将会被自动调用。 `failed` 方法接受事件实例和失败的异常作为参数：

```php
class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $podcast;
    
    public $connection = 'redis';
    
    public $queue = 'test';
    
    public $delay = 30;
    
    public $tries = 5;
    
    public $timeout = 30;

    public function __construct(Podcast $podcast)
    {
        $this->podcast = $podcast;
    }

    public function handle(AudioProcessor $processor)
    {
        // Process uploaded podcast...
        
        if (false) {
            $this->release(30);
        }
    }
    
    public function failed(OrderShipped $event, $exception)
    {
        //
    }
}

```

### 任务事件


```php
class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        //任务运行前
        Queue::before(function (JobProcessing $event) {
            // $event->connectionName
            // $event->job
            // $event->job->payload()
        });

        //任务运行后
        Queue::after(function (JobProcessed $event) {
            // $event->connectionName
            // $event->job
            // $event->job->payload()
        });
        
        //任务循环前
        Queue::looping(function () {
    		  while (DB::transactionLevel() > 0) {
              DB::rollBack();
            }
        });
        
        //任务失败后
         Queue::failing(function (JobFailed $event) {
            // $event->connectionName
            // $event->job
            // $event->exception
        });
        
        //异常发生后
        Queue::exceptionOccurred(function (JobFailed $event) {
            // $event->connectionName
            // $event->job
            // $event->exception
        });
    }
}
```

## 任务的分发

### 分发服务

- 写好任务类后，就能通过 `dispatch` 辅助函数来分发它了。唯一需要传递给 `dispatch` 的参数是这个任务类的实例：

```php
class PodcastController extends Controller
{
    public function store(Request $request)
    {
        // 创建播客...

        ProcessPodcast::dispatch($podcast);
    }
}

```

- 如果想延迟执行一个队列中的任务，可以用任务实例的 `delay` 方法。

```php
 ProcessPodcast::dispatch($podcast)
                ->delay(Carbon::now()->addMinutes(10));
```

- 通过推送任务到不同的队列，可以给队列任务分类，甚至可以控制给不同的队列分配多少任务。要指定队列的话，就调用任务实例的 `onQueue` 方法：

```php
ProcessPodcast::dispatch($podcast)->onQueue('processing');
```

- 如果使用了多个队列连接，可以将任务推到指定连接。要指定连接的话，可以在分发任务的时候使用 `onConnection` 方法：

```php
ProcessPodcast::dispatch($podcast)->onConnection('redis
');

```

这些链式的函数是在 `trait`：`Illuminate\Foundation\Bus\Dispatchable` 的基础上应用的，该 `trait` 由 `dispatch` 函数启动：

```php
trait Dispatchable
{
    public static function dispatch()
    {
        return new PendingDispatch(new static(...func_get_args()));
    }
}
```
`PendingDispatch` 类中定义了链式函数，该函数巧妙在析构函数中，析构函数自动调用全局函数 `dispatch`：

```php
class PendingDispatch
{
	public function __construct($job)
    {
        $this->job = $job;
    }


    public function onConnection($connection)
    {
        $this->job->onConnection($connection);

        return $this;
    }

    public function onQueue($queue)
    {
        $this->job->onQueue($queue);

        return $this;
    }

    public function delay($delay)
    {
        $this->job->delay($delay);

        return $this;
    }

    public function __destruct()
    {
        dispatch($this->job);
    }


}
```
各个函数里面的 `onConnection`、`delay`、`onQueue` 等函数是任务中的 `trait`：`Illuminate\Bus\Queueable`

```php
trait Queueable
{
	public function onConnection($connection)
    {
        $this->connection = $connection;

        return $this;
    }

    public function onQueue($queue)
    {
        $this->queue = $queue;

        return $this;
    }

    public function delay($delay)
    {
        $this->delay = $delay;

        return $this;
    }
}

```

### dispatch 任务分发源码

任务的分发离不开 `Bus` 服务，可以利用全局函数 `dispatch`，还可以使用 `Dispatchable` 这个 `trait`:

```php
class Dispatcher implements QueueingDispatcher
{
	 public function dispatch($command)
    {
        if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
            return $this->dispatchToQueue($command);
        } else {
            return $this->dispatchNow($command);
        }
    }

    protected function commandShouldBeQueued($command)
    {
        return $command instanceof ShouldQueue;
    }
}

```

我们这里主要看异步的任务：

```php
public function dispatchToQueue($command)
{
    $connection = isset($command->connection) ? $command->connection : null;

    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    } else {
        return $this->pushCommandToQueue($queue, $command);
    }
}

```

进行任务分发之前，首先要利用 `queueResolver` 连接底层驱动。如果任务类中含有 `queue` 函数，那么就会利用用户自己的 `queue` 对驱动进行推送任务。否则就会启动默认的程序：

```php
protected function pushCommandToQueue($queue, $command)
{
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    return $queue->push($command);
}

```

我们以 `redis` 为例，`queue` 这个类就是 `Illuminate\Queue\RedisQueue`:

```php
class RedisQueue extends Queue implements QueueContract
{
	public function push($job, $data = '', $queue = null)
    {
        return $this->pushRaw($this->createPayload($job, $data), $queue);
    }
    
    public function pushOn($queue, $job, $data = '')
    {
        return $this->push($job, $data, $queue);
    }

    public function later($delay, $job, $data = '', $queue = null)
    {
        return $this->laterRaw($delay, $this->createPayload($job, $data), $queue);
    }

    public function laterOn($queue, $delay, $job, $data = '')
    {
        return $this->later($delay, $job, $data, $queue);
    }
}

```

我们先看 `push`，`push` 函数调用 `pushRaw`，在调用之前，要把任务类进行序列化，并且以特定的格式进行 `json` 序列化：

```php
protected function createPayload($job, $data = '', $queue = null)
{
    $payload = json_encode($this->createPayloadArray($job, $data, $queue));

    if (JSON_ERROR_NONE !== json_last_error()) {
        throw new InvalidPayloadException;
    }

    return $payload;
}

protected function createPayloadArray($job, $data = '', $queue = null)
{
    return is_object($job)
                ? $this->createObjectPayload($job)
                : $this->createStringPayload($job, $data);
}

protected function createObjectPayload($job)
{
    return [
        'job' => 'Illuminate\Queue\CallQueuedHandler@call',
        'maxTries' => isset($job->tries) ? $job->tries : null,
        'timeout' => isset($job->timeout) ? $job->timeout : null,
        'data' => [
            'commandName' => get_class($job),
            'command' => serialize(clone $job),
        ],
    ];
}

protected function createStringPayload($job, $data)
{
    return ['job' => $job, 'data' => $data];
}
```

格式化数据之后，就会将 `json` 推送到 `redis` 队列中，对于非延时的任务，直接调用 `rpush` 即可：
 
```php    
public function pushRaw($payload, $queue = null, array $options = [])
{
    $this->getConnection()->rpush($this->getQueue($queue), $payload);

    return Arr::get(json_decode($payload, true), 'id');
}

```

对于延时的任务，会调用 `laterRaw`，调用 `redis` 的有序集合 `zadd` 函数:

```php
protected function availableAt($delay = 0)
{
    return $delay instanceof DateTimeInterface
                        ? $delay->getTimestamp()
                        : Carbon::now()->addSeconds($delay)->getTimestamp();
}
    
protected function laterRaw($delay, $payload, $queue = null)
{
    $this->getConnection()->zadd(
        $this->getQueue($queue).':delayed', $this->availableAt($delay), $payload
    );

    return Arr::get(json_decode($payload, true), 'id');
}

```

这样，相关任务就会被分发到 `redis` 对应的队列中去。

