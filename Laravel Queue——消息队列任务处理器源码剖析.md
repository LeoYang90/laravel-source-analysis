# Laravel Queue——消息队列任务处理器源码剖析

## 运行队列处理器

### 队列处理器的设置

`Laravel` 包含一个队列处理器，当新任务被推到队列中时它能处理这些任务。你可以通过 `queue:work` 命令来运行处理器。要注意，一旦 `queue:work` 命令开始，它将一直运行，直到你手动停止或者你关闭控制台：

```php
php artisan queue:work
```

- 可以指定队列处理器所使用的连接。

```php
php artisan queue:work redis

```

- 可以自定义队列处理器，方式是处理给定连接的特定队列。

```php
php artisan queue:work redis --queue=emails
```

- 可以使用 `--once` 选项来指定仅对队列中的单一任务进行处理：

```php
php artisan queue:work --once
```

- 如果一个任务失败了，会被放入延时队列中取，`--delay` 选项可以设置失败任务的延时时间：

```php
php artisan queue:work --delay=2
```

- 如果想要限制一个任务的内存，可以使用 `--memory`:

```php
php artisan queue:work --memory=128

```
- 当队列需要处理任务时，进程将继续处理任务，它们之间没有延迟。但是，如果没有新的工作可用，`--sleep` 参数决定了工作进程将 「睡眠」 多长时间：

```php
php artisan queue:work --sleep=3
```

- 可以指定 `Laravel` 队列处理器最多执行多长时间后就应该被关闭掉：

```php
php artisan queue:work --timeout=60
```
- 可以指定 `Laravel` 队列处理器失败任务重试的次数：

```php
php artisan queue:work --tries=60
```

可以看出来，队列处理器的设置大多数都可以由任务类进行设置，但是其中三个 `sleep`、`delay`、`memory` 只能由 `artisan` 来设置。

## WorkCommand 命令行启动

任务处理器进程的命令行模式会调用 `Illuminate\Queue\Console\WorkCommand`，这个类在初始化的时候依赖注入了 `Illuminate\Queue\Worker`:

```php
class WorkCommand extends Command
{
	protected $signature = 'queue:work
                            {connection? : The name of connection}
                            {--queue= : The queue to listen on}
                            {--daemon : Run the worker in daemon mode (Deprecated)}
                            {--once : Only process the next job on the queue}
                            {--delay=0 : Amount of time to delay failed jobs}
                            {--force : Force the worker to run even in maintenance mode}
                            {--memory=128 : The memory limit in megabytes}
                            {--sleep=3 : Number of seconds to sleep when no job is available}
                            {--timeout=60 : The number of seconds a child process can run}
                            {--tries=0 : Number of times to attempt a job before logging it failed}';
                            
    public function __construct(Worker $worker)
    {
        parent::__construct();

        $this->worker = $worker;
    }                         

    public function fire()
    {
        if ($this->downForMaintenance() && $this->option('once')) {
            return $this->worker->sleep($this->option('sleep'));
        }

        $this->listenForEvents();

        $connection = $this->argument('connection')
                        ?: $this->laravel['config']['queue.default'];

        $queue = $this->getQueue($connection);

        $this->runWorker(
            $connection, $queue
        );
    }
}

```
任务处理器启动后，会运行 `fire` 函数，在执行任务之前，程序首先会注册监听事件，主要监听任务完成与任务失败的情况：

```php
protected function listenForEvents()
{
    $this->laravel['events']->listen(JobProcessed::class, function ($event) {
        $this->writeOutput($event->job, false);
    });

    $this->laravel['events']->listen(JobFailed::class, function ($event) {
        $this->writeOutput($event->job, true);

        $this->logFailedJob($event);
    });
}

protected function writeOutput(Job $job, $failed)
{
    if ($failed) {
        $this->output->writeln('<error>['.Carbon::now()->format('Y-m-d H:i:s').'] Failed:</error> '.$job->resolveName());
    } else {
        $this->output->writeln('<info>['.Carbon::now()->format('Y-m-d H:i:s').'] Processed:</info> '.$job->resolveName());
    }
}

protected function logFailedJob(JobFailed $event)
{
    $this->laravel['queue.failer']->log(
        $event->connectionName, $event->job->getQueue(),
        $event->job->getRawBody(), $event->exception
    );
}
```
启动任务管理器 `runWorker`,该函数默认会调用 `Illuminate\Queue\Worker` 的 `daemon` 函数，只有在命令中强制 `--once` 参数的时候，才会执行 `runNestJob` 函数:

```php
protected function runWorker($connection, $queue)
{
    $this->worker->setCache($this->laravel['cache']->driver());

    return $this->worker->{$this->option('once') ? 'runNextJob' : 'daemon'}(
        $connection, $queue, $this->gatherWorkerOptions()
    );
}
```

## Worker 任务调度

![](http://owql68l6p.bkt.clouddn.com/QQ20171209-000154@2x.png)

我们接下来接着看 `daemon` 函数：

```php
public function daemon($connectionName, $queue, WorkerOptions $options)
{
    $this->listenForSignals();

    $lastRestart = $this->getTimestampOfLastQueueRestart();

    while (true) {
        if (! $this->daemonShouldRun($options)) {
            $this->pauseWorker($options, $lastRestart);

            continue;
        }

        $job = $this->getNextJob(
            $this->manager->connection($connectionName), $queue
        );

        $this->registerTimeoutHandler($job, $options);

        if ($job) {
            $this->runJob($job, $connectionName, $options);
        } else {
            $this->sleep($options->sleep);
        }

        $this->stopIfNecessary($options, $lastRestart);
    }
}

```

### 信号处理

`listenForSignals` 函数用于 PHP 7.1 版本以上，用于脚本的信号处理。所谓的信号处理，就是由 `Process Monitor`（如 `Supervisor` ）发送并与我们的脚本进行通信的异步通知。

```php
protected function listenForSignals()
{
    if ($this->supportsAsyncSignals()) {
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, function () {
            $this->shouldQuit = true;
        });

        pcntl_signal(SIGUSR2, function () {
            $this->paused = true;
        });

        pcntl_signal(SIGCONT, function () {
            $this->paused = false;
        });
    }
}

protected function supportsAsyncSignals()
{
    return version_compare(PHP_VERSION, '7.1.0') >= 0 &&
           extension_loaded('pcntl');
}
```

`pcntl_async_signals()` 被调用来启用信号处理，然后我们为多个信号注册处理程序：

- 当脚本被 `Supervisor` 指示关闭时，会引发信号 `SIGTERM`
- `SIGUSR2` 是用户定义的信号，Laravel用来表示脚本应该暂停。
- 当暂停的脚本被 `Supervisor` 指示继续进行时，会引发 `SIGCONT`

在真正运行任务之前，程序还从 `cache` 中取了一次最后一次重启的时间：

```php
protected function getTimestampOfLastQueueRestart()
{
    if ($this->cache) {
        return $this->cache->get('illuminate:queue:restart');
    }
}

```

### 确定 `worker` 是否应该处理作业

进入循环后，首先要判断当前脚本是应该处理任务，还是应该暂停，还是应该退出：

```php
protected function daemonShouldRun(WorkerOptions $options)
{
    return ! (($this->manager->isDownForMaintenance() && ! $options->force) ||
        $this->paused ||
        $this->events->until(new Events\Looping) === false);
}
```

以下几种情况，循环将不会处理任务：

- 脚本处于 `维护模式` 并且没有 `--force` 选项
- 脚本被 `supervisor` 暂停
- 脚本的 `looping` 事件监听器返回 `false`

`looping` 事件监听器在每次循环的时候都会被启动，如果返回 `false`，那么当前的循环将会被暂停：`pauseWorker`:

```php
protected function pauseWorker(WorkerOptions $options, $lastRestart)
{
    $this->sleep($options->sleep > 0 ? $options->sleep : 1);

    $this->stopIfNecessary($options, $lastRestart);
}
```

脚本在 `sleep` 一段时间之后，就要重新判断当前脚本是否需要 `stop`：

```php
protected function stopIfNecessary(WorkerOptions $options, $lastRestart)
{
    if ($this->shouldQuit) {
        $this->kill();
    }

    if ($this->memoryExceeded($options->memory)) {
        $this->stop(12);
    } elseif ($this->queueShouldRestart($lastRestart)) {
        $this->stop();
    }
}

protected function queueShouldRestart($lastRestart)
{
    return $this->getTimestampOfLastQueueRestart() != $lastRestart;
}

protected function getTimestampOfLastQueueRestart()
{
    if ($this->cache) {
        return $this->cache->get('illuminate:queue:restart');
    }
}
```

以下情况脚本将会被 `stop`：

- 脚本被 `supervisor` 退出
- 内存超限
- 脚本被重启过

```php
public function kill($status = 0)
{
    if (extension_loaded('posix')) {
        posix_kill(getmypid(), SIGKILL);
    }

    exit($status);
}

public function stop($status = 0)
{
    $this->events->fire(new Events\WorkerStopping);

    exit($status);
}
```

脚本被重启，当前的进程需要退出并且重新加载。

### 获取下一个任务

当含有多个队列的时候，命令行可以用 `,` 连接多个队列的名字，位于前面的队列优先级更高：

```php
protected function getNextJob($connection, $queue)
{
    try {
        foreach (explode(',', $queue) as $queue) {
            if (! is_null($job = $connection->pop($queue))) {
                return $job;
            }
        }
    } catch (Exception $e) {
        $this->exceptions->report($e);
    } catch (Throwable $e) {
        $this->exceptions->report(new FatalThrowableError($e));
    }
}

```

`$connection` 是具体的驱动，我们这里是 `Illuminate\Queue\RedisQueue`:

```php
class RedisQueue extends Queue implements QueueContract
{
	public function pop($queue = null)
    {
        $this->migrate($prefixed = $this->getQueue($queue));

        list($job, $reserved) = $this->retrieveNextJob($prefixed);

        if ($reserved) {
            return new RedisJob(
                $this->container, $this, $job,
                $reserved, $this->connectionName, $queue ?: $this->default
            );
        }
    }
】

protected function getQueue($queue)
{
    return 'queues:'.($queue ?: $this->default);
}
```

在从队列中取出任务之前，需要先将 `delay` 队列和 `reserved` 队列中已经到时间的任务放到主队列中：

```php
protected function migrate($queue)
{
    $this->migrateExpiredJobs($queue.':delayed', $queue);

    if (! is_null($this->retryAfter)) {
        $this->migrateExpiredJobs($queue.':reserved', $queue);
    }
}

public function migrateExpiredJobs($from, $to)
{
    return $this->getConnection()->eval(
        LuaScripts::migrateExpiredJobs(), 2, $from, $to, $this->currentTime()
    );
}
```

由于从队列取出任务、在队列删除任务、压入主队列是三个操作，为了防止并发，程序这里使用了 `LUA` 脚本，保证三个操作的原子性：

```php
public static function migrateExpiredJobs()
{
    return <<<'LUA'
		  -- Get all of the jobs with an expired "score"...
        local val = redis.call('zrangebyscore', KEYS[1], '-inf', ARGV[1])

        -- If we have values in the array, we will remove them from the first queue
        -- and add them onto the destination queue in chunks of 100, which moves
        -- all of the appropriate jobs onto the destination queue very safely.
        if(next(val) ~= nil) then
          redis.call('zremrangebyrank', KEYS[1], 0, #val - 1)

          for i = 1, #val, 100 do
            redis.call('rpush', KEYS[2], unpack(val, i, math.min(i+99, #val)))
          end
        end

      return val
    LUA;
}
```

接下来，就要从主队列中获取下一个任务，在取出下一个任务之后，还要将任务放入 `reserved` 队列中，当任务执行失败后，该任务会进行重试。

```php
protected function retrieveNextJob($queue)
{
    return $this->getConnection()->eval(
        LuaScripts::pop(), 2, $queue, $queue.':reserved',
        $this->availableAt($this->retryAfter)
    );
}


public static function pop()
{
    return <<<'LUA'
        -- Pop the first job off of the queue...
        local job = redis.call('lpop', KEYS[1])
        local reserved = false

        if(job ~= false) then
            -- Increment the attempt count and place job on the reserved queue...
            reserved = cjson.decode(job)
            reserved['attempts'] = reserved['attempts'] + 1
            reserved = cjson.encode(reserved)
            redis.call('zadd', KEYS[2], ARGV[1], reserved)
        end

        return {job, reserved}
    LUA;
}
```

从 `redis` 中获取到 `job` 之后，就会将其包装成 `RedisJob` 类：

```php
public function __construct(Container $container, RedisQueue $redis, $job, $reserved, $connectionName, $queue)
{
    $this->job = $job;
    $this->redis = $redis;
    $this->queue = $queue;
    $this->reserved = $reserved;
    $this->container = $container;
    $this->connectionName = $connectionName;

    $this->decoded = $this->payload();
}

public function payload()
{
    return json_decode($this->getRawBody(), true);
}

public function getRawBody()
{
    return $this->job;
}
```

### 超时处理

如果一个脚本超时， `pcntl_alarm` 将会启动并杀死当前的 `work` 进程。杀死进程后， `work` 进程将会被守护进程重启，继续进行下一个任务。

```php
protected function registerTimeoutHandler($job, WorkerOptions $options)
{
    if ($options->timeout > 0 && $this->supportsAsyncSignals()) {
        pcntl_signal(SIGALRM, function () {
            $this->kill(1);
        });

        pcntl_alarm($this->timeoutForJob($job, $options) + $options->sleep);
    }
}

protected function timeoutForJob($job, WorkerOptions $options)
{
    return $job && ! is_null($job->timeout()) ? $job->timeout() : $options->timeout;
}

```

## 任务事务

运行任务前后会启动两个事件 `JobProcessing` 与 `JobProcessed`，这两个事件需要事先注册监听者

```php
protected function runJob($job, $connectionName, WorkerOptions $options)
{
    try {
        return $this->process($connectionName, $job, $options);
    } catch (Exception $e) {
        $this->exceptions->report($e);
    } catch (Throwable $e) {
        $this->exceptions->report(new FatalThrowableError($e));
    }
}

public function process($connectionName, $job, WorkerOptions $options)
{
    try {
        $this->raiseBeforeJobEvent($connectionName, $job);

        $this->markJobAsFailedIfAlreadyExceedsMaxAttempts(
            $connectionName, $job, (int) $options->maxTries
        );

        $job->fire();

        $this->raiseAfterJobEvent($connectionName, $job);
    } catch (Exception $e) {
        $this->handleJobException($connectionName, $job, $options, $e);
    } catch (Throwable $e) {
        $this->handleJobException(
            $connectionName, $job, $options, new FatalThrowableError($e)
        );
    }
}
```

### 任务前与任务后事件

`raiseBeforeJobEvent` 函数用于触发任务处理前的事件，`raiseAfterJobEvent` 函数用于触发任务处理后的事件：

```php
protected function raiseBeforeJobEvent($connectionName, $job)
{
    $this->events->fire(new Events\JobProcessing(
        $connectionName, $job
    ));
}
    
protected function raiseAfterJobEvent($connectionName, $job)
{
    $this->events->fire(new Events\JobProcessed(
        $connectionName, $job
    ));
}

```

## 任务异常处理

![](http://owql68l6p.bkt.clouddn.com/QQ20171209-003432@2x.png)

任务在运行过程中会遇到异常情况，这个时候就要判断当前任务的失败次数是不是超过限制。如果没有超过限制，那么就会把当前任务重新放回队列当中；如果超过了限制，那么就要标记当前任务为失败任务，并且将任务从 `reserved` 队列中删除。

### 任务失败

`markJobAsFailedIfAlreadyExceedsMaxAttempts` 函数用于任务运行前，判断当前任务是否重试次数超过限制:

```php      
protected function markJobAsFailedIfAlreadyExceedsMaxAttempts($connectionName, $job, $maxTries)
{
    $maxTries = ! is_null($job->maxTries()) ? $job->maxTries() : $maxTries;

    if ($maxTries === 0 || $job->attempts() <= $maxTries) {
        return;
    }

    $this->failJob($connectionName, $job, $e = new MaxAttemptsExceededException(
        'A queued job has been attempted too many times. The job may have previously timed out.'
    ));

    throw $e;
}

public function maxTries()
{
    return array_get($this->payload(), 'maxTries');
}

public function attempts()
{
    return Arr::get($this->decoded, 'attempts') + 1;
}

protected function failJob($connectionName, $job, $e)
{
    return FailingJob::handle($connectionName, $job, $e);
}
```

当遇到重试次数大于限制的任务，`work` 进程就会调用 `FailingJob`:

```php
protected function failJob($connectionName, $job, $e)
{
    return FailingJob::handle($connectionName, $job, $e);
}
    
public static function handle($connectionName, $job, $e = null)
{
    $job->markAsFailed();

    if ($job->isDeleted()) {
        return;
    }

    try {
        $job->delete();

        $job->failed($e);
    } finally {
        static::events()->fire(new JobFailed(
            $connectionName, $job, $e ?: new ManuallyFailedException
        ));
    }
}

public function markAsFailed()
{
    $this->failed = true;
}

public function delete()
{
    parent::delete();

    $this->redis->deleteReserved($this->queue, $this);
}

public function isDeleted()
{
    return $this->deleted;
}
```
`FailingJob` 会标记当前任务 `failed`、`deleted`，并且会将当前任务移除 `reserved` 队列，不会再重试：

```php
public function deleteReserved($queue, $job)
{
    $this->getConnection()->zrem($this->getQueue($queue).':reserved', $job->getReservedJob());
}

```

`FailingJob` 还会调用 `RedisJob` 的 `failed` 函数，并且触发 `JobFailed` 事件：

```php
public function failed($e)
{
    $this->markAsFailed();

    $payload = $this->payload();

    list($class, $method) = JobName::parse($payload['job']);

    if (method_exists($this->instance = $this->resolve($class), 'failed')) {
        $this->instance->failed($payload['data'], $e);
    }
}

```
程序会解析 `job` 类，我们先前在 `redis` 中已经存储了：

```php
[
    'job' => 'Illuminate\Queue\CallQueuedHandler@call',
    'maxTries' => isset($job->tries) ? $job->tries : null,
    'timeout' => isset($job->timeout) ? $job->timeout : null,
    'data' => [
        'commandName' => get_class($job),
        'command' => serialize(clone $job),
    ],
];

```

我们接着看 `failed` 函数：

```php
public function failed(array $data, $e)
{
    $command = unserialize($data['command']);

    if (method_exists($command, 'failed')) {
        $command->failed($e);
    }
}

```

可以看到，最后程序调用了任务类的 `failed` 函数。

### 异常处理

当任务遇到异常的时候，程序仍然会判断当前任务的重试次数，如果本次任务的重试次数已经大于或等于限制，那么就会停止重试，标记为失败；否则就会重新放入队列，记录日志。

```php
protected function handleJobException($connectionName, $job, WorkerOptions $options, $e)
{
    try {
        $this->markJobAsFailedIfWillExceedMaxAttempts(
            $connectionName, $job, (int) $options->maxTries, $e
        );

        $this->raiseExceptionOccurredJobEvent(
            $connectionName, $job, $e
        );
    } finally {
        if (! $job->isDeleted()) {
            $job->release($options->delay);
        }
    }

    throw $e;
}

protected function markJobAsFailedIfWillExceedMaxAttempts($connectionName, $job, $maxTries, $e)
{
    $maxTries = ! is_null($job->maxTries()) ? $job->maxTries() : $maxTries;

    if ($maxTries > 0 && $job->attempts() >= $maxTries) {
        $this->failJob($connectionName, $job, $e);
    }
}

public function release($delay = 0)
{
    parent::release($delay);

    $this->redis->deleteAndRelease($this->queue, $this, $delay);
}

public function deleteAndRelease($queue, $job, $delay)
{
    $queue = $this->getQueue($queue);

    $this->getConnection()->eval(
        LuaScripts::release(), 2, $queue.':delayed', $queue.':reserved',
        $job->getReservedJob(), $this->availableAt($delay)
    );
}
```

一旦任务出现异常错误。那么该任务将会立刻从 `reserved` 队列放入 `delayed` 队列，并且抛出异常，抛出异常后，程序会将其记录在日志中。

```php
public static function release()
{
    return <<<'LUA'
        -- Remove the job from the current queue...
        redis.call('zrem', KEYS[2], ARGV[1])

        -- Add the job onto the "delayed" queue...
        redis.call('zadd', KEYS[1], ARGV[2], ARGV[1])

        return true
    LUA;
}

```

## 任务的运行

任务的运行首先会调用 `CallQueuedHandler` 的 `call` 函数：

```php
public function fire()
{
    $payload = $this->payload();

    list($class, $method) = JobName::parse($payload['job']);

    with($this->instance = $this->resolve($class))->{$method}($this, $payload['data']);
}

public function call(Job $job, array $data)
{
    $command = $this->setJobInstanceIfNecessary(
        $job, unserialize($data['command'])
    );

    $this->dispatcher->dispatchNow(
        $command, $handler = $this->resolveHandler($job, $command)
    );

    if (! $job->isDeletedOrReleased()) {
        $job->delete();
    }
}
```

`setJobInstanceIfNecessary` 函数用于为任务类的 `trait`: `InteractsWithQueue` 的设置任务类：

```php
protected function setJobInstanceIfNecessary(Job $job, $instance)
{
    if (in_array(InteractsWithQueue::class, class_uses_recursive(get_class($instance)))) {
        $instance->setJob($job);
    }

    return $instance;
}

trait InteractsWithQueue
{
	public function setJob(JobContract $job)
    {
        $this->job = $job;

        return $this;
    }
}

```

接着任务的运行就要交给 `dispatch` :

```php
public function dispatchNow($command, $handler = null)
{
    if ($handler || $handler = $this->getCommandHandler($command)) {
        $callback = function ($command) use ($handler) {
            return $handler->handle($command);
        };
    } else {
        $callback = function ($command) {
            return $this->container->call([$command, 'handle']);
        };
    }

    return $this->pipeline->send($command)->through($this->pipes)->then($callback);
}

public function getCommandHandler($command)
{
    if ($this->hasCommandHandler($command)) {
        return $this->container->make($this->handlers[get_class($command)]);
    }

    return false;
}

public function hasCommandHandler($command)
{
    return array_key_exists(get_class($command), $this->handlers);
}
```

如果不对 `dispatcher` 类进行任何 `map` 函数设置，`getCommandHandler` 将会返回 `null`，此时就会调用任务类的 `handle` 函数，进行具体的业务逻辑。

任务结束后，就会调用 `delete` 函数：

```php
public function delete()
{
    parent::delete();

    $this->redis->deleteReserved($this->queue, $this);
}

public function deleteReserved($queue, $job)
{
    $this->getConnection()->zrem($this->getQueue($queue).':reserved', $job->getReservedJob());
}
```

这样，运行成功的任务会从 `reserved` 中删除。