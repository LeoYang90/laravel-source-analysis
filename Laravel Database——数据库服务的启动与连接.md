## **前言**
数据库是 `laravel` 及其重要的组成部分，大致的讲，`laravel` 的数据库功能可以分为两部分：数据库 `DB`、数据库 `Eloquent Model`。数据库的 `Eloquent` 是功能十分丰富的 `ORM`，让我们可以避免写繁杂的 `sql` 语句。数据库 `DB` 是比较底层的与 `pdo` 交互的功能，`Eloquent` 的底层依赖于 `DB`。本文将会介绍数据库 `DB` 中关于数据库服务的启动与连接部分。

在详细讲解数据库各个功能之前，我们先看看支撑着整个 `laravel` 数据库功能的框架：

![Markdown](http://owql68l6p.bkt.clouddn.com/17-9-23/18341131.jpg)

- `DB` 也就是 `DatabaseManager`，承担着数据库接口的工作，一切数据库相关的操作，例如查询、更新、插入、删除都可以通过 `DB` 这个接口来完成。但是，具体的调用 `pdo` API 的工作却不是由该类完成的，它仅仅是一个对外的接口而已。
- `ConnectionFactory` 顾名思义专门为 `DB` 构造初始化 `connector`、`connection` 对象， 
- `connector` 负责数据库的连接功能，为保障程序的高效，`laravel` 将其包装成为闭包函数，并将闭包函数作为 `connection` 的一个成员对象，实现懒加载。
- `connection` 负责数据库的具体功能，负责底层与 `pdo` API 的交互。

 
## **数据库服务的注册与启动**

数据库服务也是一种服务提供者：`Illuminate\Database\DatabaseServiceProvider`

```php
public function register()
{
    Model::clearBootedModels();

    $this->registerConnectionServices();

    $this->registerEloquentFactory();

    $this->registerQueueableEntityResolver();
}
```
我们先来看这个注册函数的第一句: `Model::clearBootedModels()`。这一句其实是为了 `Eloquent` 服务的启动做准备。数据库的 `Eloquent Model` 有一个静态的成员变量数组 `$booted`，这个静态数组存储了所有已经被初始化的数据库 `model` ，以便加载数据库模型时更加迅速。因此，在 `Eloquent` 服务启动之前需要初始化静态成员变量 `$booted`：

```php
public static function clearBootedModels()
{
    static::$booted = [];

    static::$globalScopes = [];
}
```

接下来我们就开始看数据库服务的注册最重要的两部分：`ConnectionServices` 与 `Eloquent`.

### **ConnectionServices 注册**

```php
protected function registerConnectionServices()
{
    $this->app->singleton('db.factory', function ($app) {
        return new ConnectionFactory($app);
    });

   $this->app->singleton('db', function ($app) {
        return new DatabaseManager($app, $app['db.factory']);
    });

    $this->app->bind('db.connection', function ($app) {
        return $app['db']->connection();
    });
}
```
可以看出，数据库服务向 `IOC` 容器注册了 `db`、`db.factory` 与 `db.connection`。

- 最重要的莫过于 `db` 对象，它有一个 `Facade` 是 `DB`， 我们可以利用 `DB::connection()` 来连接任意数据库，可以利用 `DB::select()` 来进行数据库的查询，可以说 `DB` 就是我们操作数据库的接口。
- `db.factory` 负责为 `DB` 创建 `connector` 提供数据库的底层连接服务，负责为 `DB` 创建 `connection` 对象来进行数据库的查询等操作。
- `db.cønnection` 是 `laravel` 用于与数据库 `pdo` 接口进行交互的底层类，可用于数据库的查询、更新、创建等操作。

### **Eloquent 注册**

```php
protected function registerEloquentFactory()
{
    $this->app->singleton(FakerGenerator::class, function () {
        return FakerFactory::create();
    });

    $this->app->singleton(EloquentFactory::class, function ($app) {
        return EloquentFactory::construct(
            $app->make(FakerGenerator::class), database_path('factories')
        );
    });
}
```
`EloquentFactory` 用于创建 `Eloquent Model`，用于全局函数 `factory()` 来创建数据库模型。

### **数据库服务的启动**

```php
public function boot()
{
    Model::setConnectionResolver($this->app['db']);

    Model::setEventDispatcher($this->app['events']);
}
```
数据库服务的启动主要设置 `Eloquent Model` 的 `connection resolver`，用于数据库模型 `model` 利用 `db` 来来连接数据库。
还有设置数据库事件的分发器 `dispatcher`，用于监听数据库的事件。

 
## **DatabaseManager——数据库的接口**

如果我们想要使用任何数据库服务，首先要做的事情当然是利用用户名与密码来连接数据库。在 `laravel` 中，数据库的用户名与密码一般放在 `.env` 文件中或者放入 `nginx` 配置中，并且利用数据库的接口 `DB` 来与 `pdo` 进行交互，利用 `pdo` 来连接数据库。

`DB` 即是类 `Illuminate\Database\DatabaseManager`，首先我们来看看其构造函数：

```php
public function __construct($app, ConnectionFactory $factory)
{
    $this->app = $app;
    $this->factory = $factory;
}
``` 
我们称 `DB` 为一个接口，或者是一个门面模式，是因为数据库操作，例如数据库的连接或者查询、更新等操作均不是 `DB` 的功能，数据库的连接使用类 `Illuminate\Database\Connectors\Connector` 完成，数据库的查询等操作由类 `Illuminate\Database\Connection` 完成，

因此，我们不必直接操作 `connector` 或者 `connection`，仅仅会操作 `DB` 即可。

那么 `DB` 是如何实现 `connector` 或者 `connection` 的功能的呢？关键还是这个 `ConnectionFactory` 类，这个工厂类专门为 `DB` 来生成 `connection` 对象，并将其放入 `DB` 的成员变量数组 `$connections` 中去。`connection` 中会包含 `connector` 对象来实现数据库的连接工作。

```php
class DatabaseManager implements ConnectionResolverInterface
{
	protected $app;
	protected $factory;
	protected $connections = [];
	
	public function __call($method, $parameters)
    {
        return $this->connection()->$method(...$parameters);
    }
}
```
魔术函数实现了 `DB` 与 `connection` 的无缝连接，任何对数据库的操作，例如 `DB::select()`、`DB::table('user')->save()`，都会被转移至 `connection` 中去。

### **connection 函数——获取数据库连接对象**

```php
public function connection($name = null)
{
    list($database, $type) = $this->parseConnectionName($name);

    $name = $name ?: $database;

    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->configure(
                $connection = $this->makeConnection($database), $type
        );
    }

    return $this->connections[$name];
}
```

具体流程如下：

![Markdown](http://owql68l6p.bkt.clouddn.com/17-9-23/11296035.jpg)

`DB` 的 `connection` 函数可以传入数据库的名字，也可以不传任何参数，此时会连接默认数据库，默认数据库的设置在 `config/database` 文件中。

`connection` 函数流程：

- 解析数据库名称与数据库类型，例如只读、写
- 若没有创建过与该数据库的连接，则开始创建数据库连接
- 返回数据库连接对象 `connection`

```php
protected function parseConnectionName($name)
{
    $name = $name ?: $this->getDefaultConnection();

    return Str::endsWith($name, ['::read', '::write'])
                        ? explode('::', $name, 2) : [$name, null];
}

public function getDefaultConnection()
{
    return $this->app['config']['database.default'];
}
```

可以看出，若没有特别指定连接的数据库名称，那么就会利用文件 `config/database` 文件中设置的 `default` 数据库名称作为默认连接数据库名称。若数据库支持读写分离，那么还可以指定数据库的读写属性，例如 `mysql::read`。

### **makeConnection 函数——创建新的数据库连接对象**

当框架从未连接过当前数据库的时候，就要对数据库进行连接操作，首先程序会调用 `makeConnection` 函数：

```php
protected function makeConnection($name)
{
    $config = $this->configuration($name);

    if (isset($this->extensions[$name])) {
        return call_user_func($this->extensions[$name], $config, $name);
    }

    if (isset($this->extensions[$driver = $config['driver']])) {
        return call_user_func($this->extensions[$driver], $config, $name);
    }

    return $this->factory->make($config, $name);
}

```
可以看出，连接数据库仅仅需要两个步骤：获取数据库配置、利用 `connection factory` 获取 `connection` 对象。

获取数据库配置：

```php
protected function configuration($name)
{
    $name = $name ?: $this->getDefaultConnection();

    $connections = $this->app['config']['database.connections'];

    if (is_null($config = Arr::get($connections, $name))) {
        throw new InvalidArgumentException("Database [$name] not configured.");
    }

    return $config;
}
```
也是非常简单，直接从配置文件中获取当前数据库的配置：

```php
'connections' => [
    'mysql' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'database' => env('DB_DATABASE', 'forge'),
        'username' => env('DB_USERNAME', 'forge'),
        'password' => env('DB_PASSWORD', ''),
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => '',
        'strict' => true,
        'engine' => null,
        'read' => [
            'database'  => env('DB_DATABASE', 'forge'),
        ],
        'write' => [
            'database'  => env('DB_DATABASE', 'forge'),
        ],
    ],
],
```

`$this->factory->make($config, $name)` 函数向我们提供了数据库连接对象。

### **configure——连接对象读写配置**

当我们从 `connection factory` 中获取到连接对象 `connection` 之后，我们就要根据传入的参数进行读写配置：

```php
protected function configure(Connection $connection, $type)
{
    $connection = $this->setPdoForType($connection, $type);

    if ($this->app->bound('events')) {
        $connection->setEventDispatcher($this->app['events']);
    }

    $connection->setReconnector(function ($connection) {
        $this->reconnect($connection->getName());
    });

    return $connection;
}
```

`setPdoForType` 函数就是根据 `type` 来设置读写：

当我们需要 `read` 数据库连接时，我们将 `read-pdo` 设置为主 `pdo`。当我们需要 `write` 数据库连接时，我们将读写 `pdo` 都设置为 `write-pdo`：  

```php
protected function setPdoForType(Connection $connection, $type = null)
{
    if ($type == 'read') {
        $connection->setPdo($connection->getReadPdo());
    } elseif ($type == 'write') {
        $connection->setReadPdo($connection->getPdo());
    }

    return $connection;
}
```
 
## **ConnectionFactory——数据库连接对象工厂**

![Markdown](http://owql68l6p.bkt.clouddn.com/17-9-23/26227209.jpg)

### **make 函数——工厂接口**

获取到了数据库的配置参数之后，就要利用 `ConnectionFactory` 来获取 `connection` 对象了：

```php
public function make(array $config, $name = null)
{
    $config = $this->parseConfig($config, $name);

    if (isset($config['read'])) {
        return $this->createReadWriteConnection($config);
    }

    return $this->createSingleConnection($config);
}

protected function parseConfig(array $config, $name)
{
    return Arr::add(Arr::add($config, 'prefix', ''), 'name', $name);
}
```

在建立连接之前，要先向配置参数中添加默认的 `prefix` 属性与 `name` 属性。

接着，就要判断我们在配置文件中是否设置了读写分离。如果设置了读写分离，那么就会调用 `createReadWriteConnection` 函数，生成具有读、写两个功能的 `connection`；否则的话，就会调用 `createSingleConnection` 函数，生成普通的连接对象。

### **createSingleConnection 函数——制造数据库连接对象**

`createSingleConnection` 函数是类 `ConnectionFactory` 的核心，用于生成新的数据库连接对象。

```php
protected function createSingleConnection(array $config)
{
    $pdo = $this->createPdoResolver($config);

    return $this->createConnection(
        $config['driver'], $pdo, $config['database'], $config['prefix'], $config
    );
}
```

`ConnectionFactory` 也很简单，只做了两件事情：制造 `pdo` 连接的闭包函数、构造一个新的 `connection` 对象。

### **createPdoResolver——数据库连接器闭包函数**

根据配置参数中是否含有 `host`，创建不同的闭包函数：

```php
protected function createPdoResolver(array $config)
{
    return array_key_exists('host', $config)
                        ? $this->createPdoResolverWithHosts($config)
                        : $this->createPdoResolverWithoutHosts($config);
}
```
不带有 `host` 的 `pdo` 闭包函数：

```php
protected function createPdoResolverWithoutHosts(array $config)
{
    return function () use ($config) {
        return $this->createConnector($config)->connect($config);
    };
}
```
可以看出，不带有 `pdo` 的闭包函数非常简单，仅仅创建 `connector` 对象，利用 `connector` 对象进行数据库的连接。

带有 `host` 的 `pdo` 闭包函数：

```php
protected function createPdoResolverWithHosts(array $config)
{
    return function () use ($config) {
        foreach (Arr::shuffle($hosts = $this->parseHosts($config)) as $key => $host) {
            $config['host'] = $host;

            try {
                return $this->createConnector($config)->connect($config);
            } catch (PDOException $e) {
                if (count($hosts) - 1 === $key && $this->container->bound(ExceptionHandler::class)) {
                    $this->container->make(ExceptionHandler::class)->report($e);
                }
            }
        }

        throw $e;
    };
}

protected function parseHosts(array $config)
{
    $hosts = array_wrap($config['host']);

    if (empty($hosts)) {
        throw new InvalidArgumentException('Database hosts array is empty.');
    }

    return $hosts;
}
```
带有 `host` 的闭包函数相对比较复杂，首先程序会随机选择不同的数据库依次来建立数据库连接，若均失败，就会报告异常。

### **createConnector——创建连接器**

程序会根据配置参数中 `driver` 的不同来创建不同的连接器，每个连接器都继承自 `connector` 类，用于连接数据库。 

```php
public function createConnector(array $config)
{
    if (! isset($config['driver'])) {
        throw new InvalidArgumentException('A driver must be specified.');
    }

    if ($this->container->bound($key = "db.connector.{$config['driver']}")) {
        return $this->container->make($key);
    }

    switch ($config['driver']) {
        case 'mysql':
            return new MySqlConnector;
        case 'pgsql':
            return new PostgresConnector;
        case 'sqlite':
            return new SQLiteConnector;
        case 'sqlsrv':
            return new SqlServerConnector;
    }

    throw new InvalidArgumentException("Unsupported driver [{$config['driver']}]");
}
```

### **createConnection——创建连接对象**

```php
protected function createConnection($driver, $connection, $database, $prefix = '', array $config = [])
{
    if ($resolver = Connection::getResolver($driver)) {
        return $resolver($connection, $database, $prefix, $config);
    }

    switch ($driver) {
        case 'mysql':
            return new MySqlConnection($connection, $database, $prefix, $config);
        case 'pgsql':
            return new PostgresConnection($connection, $database, $prefix, $config);
        case 'sqlite':
            return new SQLiteConnection($connection, $database, $prefix, $config);
        case 'sqlsrv':
            return new SqlServerConnection($connection, $database, $prefix, $config);
    }

    throw new InvalidArgumentException("Unsupported driver [$driver]");
}
```

创建 `pdo` 闭包函数之后，会将该闭包函数放入 `connection` 对象当中去。以后我们利用 `connection` 对象进行查询或者更新数据库时，程序便会运行该闭包函数，与数据库进行连接。

### **createReadWriteConnection——创建读写连接对象**

当配置文件中有 `read`、`write` 等配置项时，说明用户希望创建一个可以读写分离的数据库连接，此时：
 
```php
protected function createReadWriteConnection(array $config)
{
    $connection = $this->createSingleConnection($this->getWriteConfig($config));

    return $connection->setReadPdo($this->createReadPdo($config));
}

protected function getWriteConfig(array $config)
{
    return $this->mergeReadWriteConfig(
        $config, $this->getReadWriteConfig($config, 'write')
    );
}

protected function getReadWriteConfig(array $config, $type)
{
    return isset($config[$type][0])
                    ? $config[$type][array_rand($config[$type])]
                    : $config[$type];
}

protected function mergeReadWriteConfig(array $config, array $merge)
{
    return Arr::except(array_merge($config, $merge), ['read', 'write']);
}
```
可以看出，程序先读出关于 `write` 数据库的配置，之后将其合并到总配置当中，删除关于 `read` 数据库的配置，然后进行 `createSingleConnection` 建立新的连接对象。

建立连接对象之后，再根据 `read` 数据库的配置，生成 `read` 数据库的 `pdo` 闭包函数，并调用 `setReadPdo` 将其设置为读库 `pdo`。
 
```php
protected function createReadPdo(array $config)
{
    return $this->createPdoResolver($this->getReadConfig($config));
}

protected function getReadConfig(array $config)
{
    return $this->mergeReadWriteConfig(
        $config, $this->getReadWriteConfig($config, 'read')
    );
}
```


 
## **connector 连接**

我们以 `mysql` 为例：

```php
class MySqlConnector extends Connector implements ConnectorInterface 
{
	public function connect(array $config)
	{
    	$dsn = $this->getDsn($config);

    	$options = $this->getOptions($config);

    	$connection = $this->createConnection($dsn, $config, $options);

   		if (! empty($config['database'])) {
       	 $connection->exec("use `{$config['database']}`;");
    	}

    	$this->configureEncoding($connection, $config);

    	$this->configureTimezone($connection, $config);

   		$this->setModes($connection, $config);

      return $connection;
    }
}
```

### **getDsn——获取数据库连接DSN参数**

```php
protected function getDsn(array $config)
{
    return $this->hasSocket($config)
                        ? $this->getSocketDsn($config)
                        : $this->getHostDsn($config);
}

protected function hasSocket(array $config)
{
    return isset($config['unix_socket']) && ! empty($config['unix_socket']);
}

protected function getSocketDsn(array $config)
{
    return "mysql:unix_socket={$config['unix_socket']};dbname={$config['database']}";
}

protected function getHostDsn(array $config)
{
    extract($config, EXTR_SKIP);

    return isset($port)
                ? "mysql:host={$host};port={$port};dbname={$database}"
                : "mysql:host={$host};dbname={$database}";
}

```

`mysql` 数据库的连接有两种：tcp连接与socket连接。

`socket` 连接更快，但是它要求应用程序与数据库在同一台机器，更普通的是使用 `tcp` 的方式连接数据库。框架根据配置参数来选择是采用 `socket` 还是 `tcp` 的方式连接数据库。

### **getOptions——pdo 属性设置**

```php
protected $options = [
    PDO::ATTR_CASE => PDO::CASE_NATURAL,
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_ORACLE_NULLS => PDO::NULL_NATURAL,
    PDO::ATTR_STRINGIFY_FETCHES => false,
    PDO::ATTR_EMULATE_PREPARES => false,
];

public function getOptions(array $config)
{
    $options = Arr::get($config, 'options', []);

    return array_diff_key($this->options, $options) + $options;
}
```

`pdo` 的属性主要有以下几种：

- `PDO::ATTR_CASE` 强制列名为指定的大小写。他的$value可为：
	- `PDO::CASE_LOWER`：强制列名小写。
	- `PDO::CASE_NATURAL`：保留数据库驱动返回的列名。
	- `PDO::CASE_UPPER`：强制列名大写。
- `PDO::ATTR_ERRMODE`：错误报告。他的$value可为：
	- `PDO::ERRMODE_SILENT`： 仅设置错误代码。
	- `PDO::ERRMODE_WARNING`: 引发 E_WARNING 错误.
	- `PDO::ERRMODE_EXCEPTION`: 抛出 exceptions 异常。
- `PDO::ATTR_ORACLE_NULLS` （在所有驱动中都可用，不仅限于Oracle）： 转换 NULL 和空字符串。他的$value可为:       
	- `PDO::NULL_NATURAL`: 不转换。 
	- `PDO::NULL_EMPTY_STRING`： 将空字符串转换成 NULL 。
	- `PDO::NULL_TO_STRING`: 将 NULL 转换成空字符串。
- `PDO::ATTR_STRINGIFY_FETCHES`: 提取的时候将数值转换为字符串。
- `PDO::ATTR_EMULATE_PREPARES` 启用或禁用预处理语句的模拟。 有些驱动不支持或有限度地支持本地预处理。使用此设置强制PDO总是模拟预处理语句（如果为 TRUE ），或试着使用本地预处理语句（如果为 `FALSE` ）。如果驱动不能成功预处理当前查询，它将总是回到模拟预处理语句上。 需要 `bool` 类型。
- `PDO::ATTR_AUTOCOMMIT`：设置当前连接 `Mysql` 服务器的客户端的SQL语句是否自动执行，默认是自动提交.
- `PDO::ATTR_PERSISTENT`：当前对Mysql服务器的连接是否是长连接.

### **createConnection——创建数据库连接**

```php
public function createConnection($dsn, array $config, array $options)
{
    list($username, $password) = [
        Arr::get($config, 'username'), Arr::get($config, 'password'),
    ];

    try {
        return $this->createPdoConnection(
            $dsn, $username, $password, $options
        );
    } catch (Exception $e) {
        return $this->tryAgainIfCausedByLostConnection(
            $e, $dsn, $username, $password, $options
        );
    }
}

protected function createPdoConnection($dsn, $username, $password, $options)
{
    if (class_exists(PDOConnection::class) && ! $this->isPersistentConnection($options)) {
        return new PDOConnection($dsn, $username, $password, $options);
    }

    return new PDO($dsn, $username, $password, $options);
}
```

当 `pdo` 对象成功的建立起来后，说明我们已经与数据库成功地建立起来了一个连接，接下来我们就可以利用这个 `pdo` 对象进行查询或者更新等操作。

当创建 `pdo` 的时候抛出异常时：

```php
protected function tryAgainIfCausedByLostConnection(Exception $e, $dsn, $username, $password, $options)
{
    if ($this->causedByLostConnection($e)) {
        return $this->createPdoConnection($dsn, $username, $password, $options);
    }

    throw $e;
}

protected function causedByLostConnection(Exception $e)
{
    $message = $e->getMessage();

    return Str::contains($message, [
        'server has gone away',
        'no connection to the server',
        'Lost connection',
        'is dead or not enabled',
        'Error while sending',
        'decryption failed or bad record mac',
        'server closed the connection unexpectedly',
        'SSL connection has been closed unexpectedly',
        'Error writing data to the connection',
        'Resource deadlock avoided',
    ]);
}
```

当判断出的异常是上面几种情况时，框架会再次尝试连接数据库。

### **configureEncoding——设置字符集与校对集**

```php
protected function configureEncoding($connection, array $config)
{
    if (! isset($config['charset'])) {
        return $connection;
    }

    $connection->prepare(
        "set names '{$config['charset']}'".$this->getCollation($config)
    )->execute();
}

protected function getCollation(array $config)
{
    return ! is_null($config['collation']) ? " collate '{$config['collation']}'" : '';
}
```
如果配置参数中设置了字符集与校对集，程序会利用配置的参数对数据库进行相关设置。

所谓的字符集与校对集设置，可以参考[mysql 中 character set 与 collation 的点滴理解](http://www.360doc.com/content/11/0303/01/2588264_97631236.shtml)

### **configureTimezone——设置时间区**

```php
protected function configureTimezone($connection, array $config)~
{
    if (isset($config['timezone'])) {
        $connection->prepare('set time_zone="'.$config['timezone'].'"')->execute();
    }
}
```

### **setModes——设置 SQL_MODE 模式**

```php
protected function setModes(PDO $connection, array $config)
{
    if (isset($config['modes'])) {
        $this->setCustomModes($connection, $config);
    } elseif (isset($config['strict'])) {
        if ($config['strict']) {
            $connection->prepare($this->strictMode())->execute();
        } else {
            $connection->prepare("set session sql_mode='NO_ENGINE_SUBSTITUTION'")->execute();
        }
    }
}

protected function setCustomModes(PDO $connection, array $config)
{
    $modes = implode(',', $config['modes']);

    $connection->prepare("set session sql_mode='{$modes}'")->execute();
}

protected function strictMode()
{
    return "set session sql_mode='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'";
}
```

以下内容参考：[mysql的sql_mode设置简介](http://www.xlaoyu.info/mysql/2016/06/07/mysql-sql_mode-setting.html):

`SQL_MODE` 直接理解就是：sql的运作模式。官方的说法是：sql_mode可以影响sql支持的语法以及数据的校验执行，这使得MySQL可以运行在不同的环境中以及和其他数据库一起运作。 

想设置sql_mode有三种方式：

- 在命令行启动MySQL时添加参数 --sql-mode="modes"
- 在MySQL的配置文件（my.cnf或者my.ini）中添加一个配置sql-mode="modes"
- 运行时修改SQL mode可以通过以下命令之一：

```php
SET GLOBAL sql_mode = 'modes';
SET SESSION sql_mode = 'modes';
```

几种常见的mode介绍

- `ONLY_FULL_GROUP_BY`： 出现在 `select` 语句、`HAVING` 条件和 `ORDER BY` 语句中的列，必须是 `GROUP BY` 的列或者依赖于 `GROUP BY` 列的函数列。

- `NO_AUTO_VALUE_ON_ZERO`： 该值影响自增长列的插入。默认设置下，插入0或 `NULL` 代表生成下一个自增长值。如果用户 希望插入的值为0，而该列又是自增长的，那么这个选项就有用了。

- `STRICT_TRANS_TABLES`： 在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做限制

- `NO_ZERO_IN_DATE`： 这个模式影响了是否允许日期中的月份和日包含0。如果开启此模式，2016-01-00是不允许的，但是0000-02-01是允许的。它实际的行为受到 `strict mode` 是否开启的影响1。

- `NO_ZERO_DATE`： 设置该值，`mysql` 数据库不允许插入零日期。它实际的行为受到 `strict mode` 是否开启的影响2。

- `ERROR_FOR_DIVISION_BY_ZERO`： 在 `INSERT` 或 `UPDATE` 过程中，如果数据被零除，则产生错误而非警告。如 果未给出该模式，那么数据被零除时 `MySQL` 返回 `NULL`

- `NO_AUTO_CREATE_USER`： 禁止 `GRANT` 创建密码为空的用户

- `NO_ENGINE_SUBSTITUTION`： 如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常

- `PIPES_AS_CONCAT`： 将”||”视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似

- `ANSI_QUOTES`： 启用 `ANSI_QUOTES` 后，不能用双引号来引用字符串，因为它被解释为识别符







