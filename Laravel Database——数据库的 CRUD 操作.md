## 前言 
当 `connection` 对象构建初始化完成后，我们就可以利用 `DB` 来进行数据库的 `CRUD` ( `Create`、`Retrieve`、`Update`、`Delete`)操作。本篇文章，我们将会讲述 `laravel` 如何与 `pdo` 交互，实现基本数据库服务的原理。

## run

`laravel` 中任何数据库的操作都要经过 `run` 这个函数，这个函数作用在于重新连接数据库、记录数据库日志、数据库异常处理：

```php
protected function run($query, $bindings, Closure $callback)
{
    $this->reconnectIfMissingConnection();

    $start = microtime(true);

    try {
        $result = $this->runQueryCallback($query, $bindings, $callback);
    } catch (QueryException $e) {
        $result = $this->handleQueryException(
            $e, $query, $bindings, $callback
        );
    }

    $this->logQuery(
        $query, $bindings, $this->getElapsedTime($start)
    );

    return $result;
}
```

### 重新连接数据库 reconnect

如果当期的 `pdo` 是空，那么就会调用 `reconnector` 重新与数据库进行连接：

```php
protected function reconnectIfMissingConnection()
{
    if (is_null($this->pdo)) {
        $this->reconnect();
    }
}

public function reconnect()
{
    if (is_callable($this->reconnector)) {
        return call_user_func($this->reconnector, $this);
    }

    throw new LogicException('Lost connection and no reconnector available.');
}

```

### 运行数据库操作

数据库的 curd 操作会被包装成为一个闭包函数，作为 `runQueryCallback` 的一个参数，当运行正常时，会返回结果，如果遇到异常的话，会将异常转化为 `QueryException`，并且抛出。

```php
protected function runQueryCallback($query, $bindings, Closure $callback)
{
    try {
        $result = $callback($query, $bindings);
    }

    catch (Exception $e) {
        throw new QueryException(
            $query, $this->prepareBindings($bindings), $e
        );
    }

    return $result;
}

```

### 数据库异常处理

当 `pdo` 查询返回异常的时候，如果当前是事务进行时，那么直接返回异常，让上一层事务来处理。

如果是由于与数据库事情连接导致的异常，那么就要重新与数据库进行连接：

```php
protected function handleQueryException($e, $query, $bindings, Closure $callback)
{
    if ($this->transactions >= 1) {
        throw $e;
    }

    return $this->tryAgainIfCausedByLostConnection(
        $e, $query, $bindings, $callback
    );
}

```

与数据库失去连接：

```php
protected function tryAgainIfCausedByLostConnection(QueryException $e, $query, $bindings, Closure $callback)
{
    if ($this->causedByLostConnection($e->getPrevious())) {
        $this->reconnect();

        return $this->runQueryCallback($query, $bindings, $callback);
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
        'Transaction() on null',
        'child connection forced to terminate due to client_idle_limit',
    ]);
}
```

### 数据库日志 

```php
public function logQuery($query, $bindings, $time = null)
{
    $this->event(new QueryExecuted($query, $bindings, $time, $this));

    if ($this->loggingQueries) {
        $this->queryLog[] = compact('query', 'bindings', 'time');
    }
}

```
想要开启或关闭日志功能：

```php
public function enableQueryLog()
{
    $this->loggingQueries = true;
}

public function disableQueryLog()
{
    $this->loggingQueries = false;
}

```

## Select 查询

```php
public function select($query, $bindings = [], $useReadPdo = true)
{
    return $this->run($query, $bindings, function ($query, $bindings) use ($useReadPdo) {
        if ($this->pretending()) {
            return [];
        }

        $statement = $this->prepared($this->getPdoForSelect($useReadPdo)
                          ->prepare($query));

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $statement->execute();

        return $statement->fetchAll();
    });
}
```

数据库的查询主要有一下几个步骤：

- 获取 `$this->pdo` 成员变量，若当前未连接数据库，则进行数据库连接，获取 `pdo` 对象。
- 设置 `pdo` 数据 `fetch` 模式
- `pdo` 进行 `sql` 语句预处理，`pdo` 绑定参数
- `sql` 语句执行，并获取数据。

### getPdoForSelect 获取 pdo 对象

```php
protected function getPdoForSelect($useReadPdo = true)
{
    return $useReadPdo ? $this->getReadPdo() : $this->getPdo();
}

public function getPdo()
{
    if ($this->pdo instanceof Closure) {
        return $this->pdo = call_user_func($this->pdo);
    }

    return $this->pdo;
}

public function getReadPdo()
{
    if ($this->transactions > 0) {
        return $this->getPdo();
    }

    if ($this->getConfig('sticky') && $this->recordsModified) {
        return $this->getPdo();
    }

    if ($this->readPdo instanceof Closure) {
        return $this->readPdo = call_user_func($this->readPdo);
    }

    return $this->readPdo ?: $this->getPdo();
}
```

`getPdo` 这里逻辑比较简单，值得我们注意的是 `getReadPdo`。为了减缓数据库的压力，我们常常对数据库进行读写分离，也就是只要当写数据库这种操作发生时，才会使用写数据库，否则都会用读数据库。这种措施减少了数据库的压力，但是也带来了一些问题，那就是读写两个数据库在一定时间内会出现数据不一致的情况，原因就是写库的数据未能及时推送给读库，造成读库数据延迟的现象。为了在一定程度上解决这类问题，`laravel` 增添了 `sticky` 选项，

从程序中我们可以看出，当我们设置选项 `sticky` 为真，并且的确对数据库进行了写操作后，`getReadPdo` 会强制返回主库的连接，这样就避免了读写分离造成的延迟问题。

还有一种情况，当数据库在执行事务期间，所有的读取操作也会被强制连接主库。

### prepared 设置数据获取方式

```php
protected $fetchMode = PDO::FETCH_OBJ;
protected function prepared(PDOStatement $statement)
{
    $statement->setFetchMode($this->fetchMode);

    $this->event(new Events\StatementPrepared(
        $this, $statement
    ));

    return $statement;
}
```
`pdo` 的 `setFetchMode` 函数用于为语句设置默认的获取模式，通常模式有一下几种：

- PDO::FETCH_ASSOC       //从结果集中获取以列名为索引的关联数组。
- PDO::FETCH_NUM         //从结果集中获取一个以列在行中的数值偏移量为索引的值数组。
- PDO::FETCH_BOTH        //这是默认值，包含上面两种数组。
- PDO::FETCH_OBJ         //从结果集当前行的记录中获取其属性对应各个列名的一个对象。
- PDO::FETCH_BOUND       //使用fetch()返回TRUE，并将获取的列值赋给在bindParm()方法中指定的相应变量。
- PDO::FETCH_LAZY        //创建关联数组和索引数组，以及包含列属性的一个对象，从而可以在这三种接口中任选一种。

### pdo 的 prepare 函数

`prepare` 函数会为 `PDOStatement::execute()` 方法准备要执行的 `SQL` 语句，`SQL` 语句可以包含零个或多个命名（`:name`）或问号（`?`）参数标记，参数在SQL执行时会被替换。

不能在 `SQL` 语句中同时包含命名（`:name`）或问号（`?`）参数标记，只能选择其中一种风格。

预处理 `SQL` 语句中的参数在使用 `PDOStatement::execute()` 方法时会传递真实的参数。

之所以使用 `prepare` 函数，是因为这个函数可以防止 `SQL` 注入，并且可以加快同一查询语句的速度。关于预处理与参数绑定防止 `SQL` 漏洞注入的原理可以参考：[Web安全之SQL注入攻击技巧与防范](http://www.cnblogs.com/csniper/p/5802202.html).

### pdo 的 bindValues 函数

在调用 `pdo` 的参数绑定函数之前，`laravel` 对参数值进一步进行了优化，把时间类型的对象利用 `grammer` 的设置重新格式化，`false` 也改为0。

`pdo` 的参数绑定函数 `bindValue`，对于使用命名占位符的预处理语句，应是类似 :name 形式的参数名。对于使用问号占位符的预处理语句，应是以1开始索引的参数位置。

```php
public function prepareBindings(array $bindings)
{
    $grammar = $this->getQueryGrammar();

    foreach ($bindings as $key => $value) {
        if ($value instanceof DateTimeInterface) {
            $bindings[$key] = $value->format($grammar->getDateFormat());
        } elseif ($value === false) {
            $bindings[$key] = 0;
        }
    }

    return $bindings;
}

public function bindValues($statement, $bindings)
{
    foreach ($bindings as $key => $value) {
        $statement->bindValue(
            is_string($key) ? $key : $key + 1, $value,
            is_int($value) ? PDO::PARAM_INT : PDO::PARAM_STR
        );
    }
}
```

## insert

```php
public function insert($query, $bindings = [])
{
    return $this->statement($query, $bindings);
}

public function statement($query, $bindings = [])
{
    return $this->run($query, $bindings, function ($query, $bindings) {
        if ($this->pretending()) {
            return true;
        }

        $statement = $this->getPdo()->prepare($query);

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $this->recordsHaveBeenModified();

        return $statement->execute();
    });
}

```
这部分的代码与 `select` 非常相似，不同之处有一下几个：

- 直接获取写库的连接，不会考虑读库
- 由于不需要返回任何数据库数据，因此也不必设置 `fetchMode`。
- `recordsHaveBeenModified` 函数标志当前连接数据库已被写入。
- 不需要调用函数 `fetchAll`

```php
public function recordsHaveBeenModified($value = true)
{
    if (! $this->recordsModified) {
        $this->recordsModified = $value;
    }
}

```

## update、delete

`affectingStatement` 这个函数与上面的 `statement` 函数一致，只是最后会返回更新、删除影响的行数。

```php
public function update($query, $bindings = [])
{
    return $this->affectingStatement($query, $bindings);
}

public function delete($query, $bindings = [])
{
    return $this->affectingStatement($query, $bindings);
}

public function affectingStatement($query, $bindings = [])
{
    return $this->run($query, $bindings, function ($query, $bindings) {
        if ($this->pretending()) {
            return 0;
        }

        $statement = $this->getPdo()->prepare($query);

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $statement->execute();

        $this->recordsHaveBeenModified(
            ($count = $statement->rowCount()) > 0
        );

        return $count;
    });
}
```

## transaction 数据库事务

为保持数据的一致性，对于重要的数据我们经常使用数据库事务，`transaction` 函数接受一个闭包函数，与一个重复尝试的次数：

```php
public function transaction(Closure $callback, $attempts = 1)
{
    for ($currentAttempt = 1; $currentAttempt <= $attempts; $currentAttempt++) {
        $this->beginTransaction();

        try {
            return tap($callback($this), function ($result) {
                $this->commit();
            });
        }

        catch (Exception $e) {
            $this->handleTransactionException(
                $e, $currentAttempt, $attempts
            );
        } catch (Throwable $e) {
            $this->rollBack();

            throw $e;
        }
    }
}

```

### 开始事务

数据库事务中非常重要的成员变量是 `$this->transactions`，它标志着当前事务的进程：

```php
public function beginTransaction()
{
    $this->createTransaction();

    ++$this->transactions;

    $this->fireConnectionEvent('beganTransaction');
}

```
可以看出，当创建事务成功后，就会累加 `$this->transactions`，并且启动 `event`，创建事务：

```php
protected function createTransaction()
{
    if ($this->transactions == 0) {
        try {
            $this->getPdo()->beginTransaction();
        } catch (Exception $e) {
            $this->handleBeginTransactionException($e);
        }
    } elseif ($this->transactions >= 1 && $this->queryGrammar->supportsSavepoints()) {
        $this->createSavepoint();
    }
}
```
如果当前没有任何事务，那么就会调用 `pdo` 来开启事务。

如果当前已经在事务保护的范围内，那么就会创建 `SAVEPOINT`，实现数据库嵌套事务：

```php
protected function createSavepoint()
{
    $this->getPdo()->exec(
        $this->queryGrammar->compileSavepoint('trans'.($this->transactions + 1))
    );
}

public function compileSavepoint($name)
{
    return 'SAVEPOINT '.$name;
}
```

如果创建事务失败，那么就会调用 `handleBeginTransactionException`：

```php
protected function handleBeginTransactionException($e)
{
    if ($this->causedByLostConnection($e)) {
        $this->reconnect();

        $this->pdo->beginTransaction();
    } else {
        throw $e;
    }
}
```

如果创建事务失败是由于与数据库失去连接的话，那么就会重新连接数据库，否则就要抛出异常。

### 事务异常

事务的异常处理比较复杂，可以先看一看代码：

```php
protected function handleTransactionException($e, $currentAttempt, $maxAttempts)
{
    if ($this->causedByDeadlock($e) &&
        $this->transactions > 1) {
        --$this->transactions;

        throw $e;
    }

    $this->rollBack();

    if ($this->causedByDeadlock($e) &&
        $currentAttempt < $maxAttempts) {
        return;
    }

    throw $e;
}

protected function causedByDeadlock(Exception $e)
{
    $message = $e->getMessage();

    return Str::contains($message, [
        'Deadlock found when trying to get lock',
        'deadlock detected',
        'The database file is locked',
        'database is locked',
        'database table is locked',
        'A table in the database is locked',
        'has been chosen as the deadlock victim',
        'Lock wait timeout exceeded; try restarting transaction',
    ]);
}
```

这里可以分为四种情况：

- 单一事务，非死锁导致的异常

单一事务就是说，此时的事务只有一层，没有嵌套事务的存在。数据库的异常也不是死锁导致的，一般是由于 `sql` 语句不正确引起的。这个时候，`handleTransactionException` 会直接回滚事务，并且抛出异常到外层：

```php
try {
      return tap($callback($this), function ($result) {
          $this->commit();
      });
}
catch (Exception $e) {
     $this->handleTransactionException(
         $e, $currentAttempt, $attempts
     );
} catch (Throwable $e) {
       $this->rollBack();

       throw $e;
}
```
接到异常之后，程序会再次回滚，但是由于 `$this->transactions` 已经为 0，因此回滚直接返回，并未真正执行，之后就会抛出异常。

- 单一事务，死锁异常

有死锁导致的单一事务异常，一般是由于其他程序同时更改了数据库，这个时候，就要判断当前重复尝试的次数是否大于用户设置的 `maxAttempts`，如果小于就继续尝试，如果大于，那么就会抛出异常。

- 嵌套事务，非死锁异常

如果出现嵌套事务，例如：

```php
\DB::transaction(function(){
    ...
    //directly or indirectly call another transaction:
    \DB::transaction(function() {
        ...
        ...
    }, 2);//attempt twice
}, 2);//attempt twice

```
如果是非死锁导致的异常，那么就要首先回滚内层的事务，抛出异常到外层事务，再回滚外层事务，抛出异常，让用户来处理。也就是说，对于嵌套事务来说，内部事务异常，一定要回滚整个事务，而不是仅仅回滚内部事务。

- 嵌套事务，死锁异常

嵌套事务的死锁异常，仍然和嵌套事务非死锁异常一样，内部事务异常，一定要回滚整个事务。

但是，不同的是，`mysql` 对于嵌套事务的回滚会导致外部事务一并回滚:[InnoDB Error Handling](https://dev.mysql.com/doc/refman/5.7/en/innodb-error-handling.html)，因此这时，我们仅仅将 `$this->transactions` 减一，并抛出异常，使得外层事务回滚抛出异常即可。

### 回滚事务

如果事务内的数据库更新操作失败，那么就要进行回滚：

```php
public function rollBack($toLevel = null)
{
    $toLevel = is_null($toLevel)
                ? $this->transactions - 1
                : $toLevel;

    if ($toLevel < 0 || $toLevel >= $this->transactions) {
        return;
    }

    $this->performRollBack($toLevel);

    $this->transactions = $toLevel;

    $this->fireConnectionEvent('rollingBack');
}

```

回滚的第一件事就是要减少 `$this->transactions` 的值，标志当前事务失败。

回滚的时候仍然要判断当前事务的状态，如果当前处于嵌套事务的话，就要进行回滚到 `SAVEPOINT`，如果是单一事务的话，才会真正回滚退出事务：

```php
protected function performRollBack($toLevel)
{
    if ($toLevel == 0) {
        $this->getPdo()->rollBack();
    } elseif ($this->queryGrammar->supportsSavepoints()) {
        $this->getPdo()->exec(
            $this->queryGrammar->compileSavepointRollBack('trans'.($toLevel + 1))
        );
    }
}

public function compileSavepointRollBack($name)
{
    return 'ROLLBACK TO SAVEPOINT '.$name;
}
```

## 提交事务

提交事务比较简单，仅仅是调用 `pdo` 的 `commit` 即可。需要注意的是对于嵌套事务的事务提交，`commit` 函数仅仅更新了 `$this->transactions`，而并没有真正提交事务，原因是内层事务的提交对于 `mysql` 来说是无效的，只有外部事务的提交才能更新整个事务。
 
```php
public function commit()
{
    if ($this->transactions == 1) {
        $this->getPdo()->commit();
    }

    $this->transactions = max(0, $this->transactions - 1);

    $this->fireConnectionEvent('committed');
}

```