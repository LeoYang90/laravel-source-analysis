title: Laravel Database——查询构造器与语法编译器源码分析(中)

tags:
  - php
  - laravel
  - database
  - 源码
categories:
  - php
  - database
  - laravel
date: 2017-09-26 12:33:48
---

---
## join 语句

`join` 语句对数据库进行连接操作，`join` 函数的连接条件可以非常简单：

```php
DB::table('services')->select('*')->join('translations AS t', 't.item_id', '=', 'services.id');
```
也可以比较复杂：

```php
DB::table('users')->select('*')->join('contacts', function ($j) {
        $j->on('users.id', '=', 'contacts.id')->orOn('users.name', '=', 'contacts.name');
    });
    //select * from "users" inner join "contacts" on "users"."id" = "contacts"."id" or "users"."name" = "contacts"."name"

    $builder = $this->getBuilder();
    DB::table('users')->select('*')->from('users')->joinWhere('contacts', 'col1', function ($j) {
        $j->select('users.col2')->from('users')->where('users.id', '=', 'foo')
    });
    //select * from "users" inner join "contacts" on "col1" = (select "users"."col2" from "users" where "users"."id" = foo)
```
还可以更加复杂：

```php
DB::table('users')->select('*')->leftJoin('contacts', function ($j) {
        $j->on('users.id', '=', 'contacts.id')->where(function ($j) {
            $j->where('contacts.country', '=', 'US')->orWhere('contacts.is_partner', '=', 1);
        });
    });
    //select * from "users" left join "contacts" on "users"."id" = "contacts"."id" and ("contacts"."country" = 'US' or "contacts"."is_partner" = 1)

    DB::table('users')->select('*')->leftJoin('contacts', function ($j) {
        $j->on('users.id', '=', 'contacts.id')->where('contacts.is_active', '=', 1)->orOn(function ($j) {
            $j->orWhere(function ($j) {
                $j->where('contacts.country', '=', 'UK')->orOn('contacts.type', '=', 'users.type');
            })->where(function ($j) {
                $j->where('contacts.country', '=', 'US')->orWhereNull('contacts.is_partner');
            });
        });
    });
    //select * from "users" left join "contacts" on "users"."id" = "contacts"."id" and "contacts"."is_active" = 1 or (("contacts"."country" = 'UK' or "contacts"."type" = "users"."type") and ("contacts"."country" = 'US' or "contacts"."is_partner" is null))
```

其实 `join` 语句与 `where` 语句非常相似，将 `join` 语句的连接条件看作 `where` 的查询条件完全可以，接下来我们看看源码。

### join 语句

从上面的示例代码可以看出，`join` 函数的参数多变，第二个参数可以是列名，也有可能是闭包函数。当第二个参数是列名的时候，第三个参数可以是闭包，还可以是符号 `=`、`>=`。

```php
public function join($table, $first, $operator = null, $second = null, $type = 'inner', $where = false)
{
    $join = new JoinClause($this, $type, $table);

    if ($first instanceof Closure) {
        call_user_func($first, $join);

        $this->joins[] = $join;

        $this->addBinding($join->getBindings(), 'join');
    }

    else {
        $method = $where ? 'where' : 'on';

        $this->joins[] = $join->$method($first, $operator, $second);

        $this->addBinding($join->getBindings(), 'join');
    }

    return $this;
}

```
可以看到，程序首先新建了一个 `JoinClause` 类对象，这个类实际上继承 `queryBuilder`，也就是说 `queryBuilder` 上的很多方法它都可以直接用，例如 `where`、`whereNull`、`whereDate` 等等。

```php
class JoinClause extends Builder
{
}
```

如果第二个参数是闭包函数的话，就会像查询组一样根据查询条件更新 `$join`。

如果第二个参数是列名，那么就会调用 `on` 方法或 `where` 方法。这两个方法的区别是，`on` 方法只支持 `whereColumn`方法和 `whereNested`，也就是说只能写出 `join on col1 = col2` 这样的语句，而 `where` 方法可以传递数组、子查询等等.

```php
public function on($first, $operator = null, $second = null, $boolean = 'and')
{
    if ($first instanceof Closure) {
        return $this->whereNested($first, $boolean);
    }

    return $this->whereColumn($first, $operator, $second, $boolean);
}

public function orOn($first, $operator = null, $second = null)
{
    return $this->on($first, $operator, $second, 'or');
}

```

### grammer——compileJoins

接下来我们来看看如何编译 `join` 语句：

```php
protected function compileJoins(Builder $query, $joins)
{
    return collect($joins)->map(function ($join) use ($query) {
        $table = $this->wrapTable($join->table);

        return trim("{$join->type} join {$table} {$this->compileWheres($join)}");
    })->implode(' ');
}

```
可以看到，`JoinClause` 在编译中是作为 `queryBuild` 对象来看待的。

## union 语句

`union` 用于合并两个或多个 `SELECT` 语句的结果集。`Union` 因为要进行重复值扫描，所以效率低。如果合并没有刻意要删除重复行，那么就使用 `Union All`。

我们在 `laravel` 中可以这样使用：

```php
$query = DB::table('users')->select('*')->where('id', '=', 1);
$query->union(DB::table('users')->select('*')->where('id', '=', 2));
//(select * from `users` where `id` = 1) union (select * from `users` where `id` = 2)
```  
还可以添加多个 `union` 语句：

```php
$query = DB::table('users')->select('*')->where('id', '=', 1);
$query->union(DB::table('users')->select('*')->where('id', '=', 2));
$query->union(DB::table('users')->select('*')->where('id', '=', 3));      
//(select * from "users" where "id" = 1) union (select * from "users" where "id" = 2) union (select * from "users" where "id" = 3)
```
`union` 语句可以与 `orderBy` 相结合：

```php
$query = DB::table('users')->select('*')->where('id', '=', 1);
$query->union(DB::table('users')->select('*')->where('id', '=', 2));
$query->orderBy('id', 'desc');
//(select * from `users` where `id` = ?) union (select * from `users` where `id` = ?) order by `id` desc
```
`union` 语句可以与 `limit`、`offset` 相结合：

```php
$query = DB::table('users')->select('*');
$query->union(DB::table('users')->select('*'));
$builder->skip(5)->take(10);
//(select * from `users`) union (select * from `dogs`) limit 10 offset 5

```

### union 函数

`union` 函数比较简单：

```php
public function union($query, $all = false)
{
    if ($query instanceof Closure) {
        call_user_func($query, $query = $this->newQuery());
    }

    $this->unions[] = compact('query', 'all');

    $this->addBinding($query->getBindings(), 'union');

    return $this;
}
```

### grammer——compileUnions

语法编译器对 `union` 的处理：

```php
public function compileSelect(Builder $query)
{
    $sql = parent::compileSelect($query);

    if ($query->unions) {
        $sql = '('.$sql.') '.$this->compileUnions($query);
    }

    return $sql;
}
    
protected function compileUnions(Builder $query)
{
    $sql = '';

    foreach ($query->unions as $union) {
        $sql .= $this->compileUnion($union);
    }

    if (! empty($query->unionOrders)) {
        $sql .= ' '.$this->compileOrders($query, $query->unionOrders);
    }

    if (isset($query->unionLimit)) {
        $sql .= ' '.$this->compileLimit($query, $query->unionLimit);
    }

    if (isset($query->unionOffset)) {
        $sql .= ' '.$this->compileOffset($query, $query->unionOffset);
    }

    return ltrim($sql);
}

protected function compileUnion(array $union)
{
    $conjuction = $union['all'] ? ' union all ' : ' union ';

    return $conjuction.'('.$union['query']->toSql().')';
}

```

可以看出，`union` 的处理比较简单，都是调用 `query->toSql` 语句而已。值得注意的是，在处理 `union` 的时候，要特别处理 `order`、`limit`、`offset`。

## orderBy 语句

orderBy 语句用法很简单，可以设置多个排序字段，也可以用原生排序语句：

```php
DB::table('users')->select('*')->orderBy('email')->orderBy('age', 'desc');

DB::table('users')->select('*')->orderBy('email')->orderByRaw('age desc');
```

### orderBy 函数

如果当前查询中有 `union` 的话，排序的变量会被放入 `unionOrders` 数组中，这个数组只有在 `compileUnions` 函数中才会被编译成 `sql` 语句。否则会被放入 `orders` 数组中，这时会被 `compileOrders` 处理：
 
```php
public function orderBy($column, $direction = 'asc')
{
    $this->{$this->unions ? 'unionOrders' : 'orders'}[] = [
        'column' => $column,
        'direction' => strtolower($direction) == 'asc' ? 'asc' : 'desc',
    ];

    return $this;
}
```

### grammer——compileOrders

orderBy 的编译也很简单：

```php
protected function compileOrders(Builder $query, $orders)
{
    if (! empty($orders)) {
        return 'order by '.implode(', ', $this->compileOrdersToArray($query, $orders));
    }

    return '';
}

protected function compileOrdersToArray(Builder $query, $orders)
{
    return array_map(function ($order) {
        return ! isset($order['sql'])
                    ? $this->wrap($order['column']).' '.$order['direction']
                    : $order['sql'];
    }, $orders);
}
```

## limit offset forPage 语句

limit offset 或者 skip take 用法很简单，有趣的是，`laravel` 考虑了负数的情况：

```php
DB::select('*')->from('users')->offset(5)->limit(10);

DB::select('*')->from('users')->skip(5)->take(10);

DB::select('*')->from('users')->skip(-5)->take(-10);

DB::select('*')->from('users')->forPage(5, 10);

```

### limit offset 函数

和 orderBy 一样，如果当前查询中有 `union` 的话，limit / offset 会被放入 unionLimit / unionOffset 中，在编译 union 的时候解析：

```php
public function take($value)
{
    return $this->limit($value);
}

public function limit($value)
{
    $property = $this->unions ? 'unionLimit' : 'limit';

    if ($value >= 0) {
        $this->$property = $value;
    }

    return $this;
}

public function skip($value)
{
    return $this->offset($value);
}

public function offset($value)
{
    $property = $this->unions ? 'unionOffset' : 'offset';

    $this->$property = max(0, $value);

    return $this;
}

public function forPage($page, $perPage = 15)
{
    return $this->skip(($page - 1) * $perPage)->take($perPage);
}
```

### grammer——compileLimit compileOffset

这个不能再简单了：

```php
protected function compileLimit(Builder $query, $limit)
{
    return 'limit '.(int) $limit;
}

protected function compileOffset(Builder $query, $offset)
{
    return 'offset '.(int) $offset;
}
```

## group 语句

`groupBy` 语句的参数形式有多种： 

```php
DB::select('*')->from('users')->groupBy('email');

DB::select('*')->from('users')->groupBy('id', 'email');

DB::select('*')->from('users')->groupBy(['id', 'email']);

DB::select('*')->from('users')->groupBy(new Raw('DATE(created_at)'));
```

`groupBy` 函数很简单，仅仅是为 `$this->groups` 成员变量合并数组：

```php
public function groupBy(...$groups)
{
    foreach ($groups as $group) {
        $this->groups = array_merge(
            (array) $this->groups,
            Arr::wrap($group)
        );
    }

    return $this;
}
```
语法编译器的处理：

```php
protected function compileGroups(Builder $query, $groups)
{
    return 'group by '.$this->columnize($groups);
}
```
## having 语句

`having` 语句的用法也很简单。大致有 `having`、`orHaving`、`havingRaw`、`orHavingRaw` 这几个函数：

```php
DB::select('*')->from('users')->having('email', '>', 1);

DB::select('*')->from('users')->groupBy('email')->having('email', '>', 1);

DB::select('*')->from('users')->having('email', 1)->orHaving('email', 2);

DB::select('*')->from('users')->havingRaw('user_foo < user_bar');

DB::select('*')->from('users')->having('baz', '=', 1)->orHavingRaw('user_foo < user_bar');
```

### having 函数
`having` 函数大致与 `whereColumn` 相同：

```php
public function having($column, $operator = null, $value = null, $boolean = 'and')
{
    $type = 'Basic';

    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    if ($this->invalidOperator($operator)) {
        list($value, $operator) = [$operator, '='];
    }

    $this->havings[] = compact('type', 'column', 'operator', 'value', 'boolean');

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'having');
    }

    return $this;
}
```
`havingRaw` 函数：

```php
public function havingRaw($sql, array $bindings = [], $boolean = 'and')
{
    $type = 'Raw';

    $this->havings[] = compact('type', 'sql', 'boolean');

    $this->addBinding($bindings, 'having');

    return $this;
}
```

### grammer——compileHavings

语法编译器：

```php
protected function compileHavings(Builder $query, $havings)
{
    $sql = implode(' ', array_map([$this, 'compileHaving'], $havings));

    return 'having '.$this->removeLeadingBoolean($sql);
}

protected function compileHaving(array $having)
{
    if ($having['type'] === 'Raw') {
        return $having['boolean'].' '.$having['sql'];
    }

    return $this->compileBasicHaving($having);
}

protected function compileBasicHaving($having)
{
    $column = $this->wrap($having['column']);

    $parameter = $this->parameter($having['value']);

    return $having['boolean'].' '.$column.' '.$having['operator'].' '.$parameter;
}
```

## when / tap / unless 语句

`when` 语句可以根据条件来判断是否执行查询条件，`unless` 与 `when` 相反，第一个参数是 `false` 才会调用闭包函数执行查询，`tap` 指定 `when` 的第一参数永远为真：

```php
$callback = function ($query, $condition) {
    $this->assertEquals($condition, 'truthy');

    $query->where('id', '=', 1);
};

$default = function ($query, $condition) {
    $this->assertEquals($condition, 0);

    $query->where('id', '=', 2);
};

DB::select('*')->from('users')->when('truthy', $callback, $default)->where('email', 'foo');

DB::select('*')->from('users')->tap($callback)->where('email', 'foo');

DB::select('*')->from('users')->unless('truthy', $callback, $default)->where('email', 'foo');
```

`when`、`unless`、`tap` 函数的实现：

```php
public function when($value, $callback, $default = null)
{
    if ($value) {
        return $callback($this, $value) ?: $this;
    } elseif ($default) {
        return $default($this, $value) ?: $this;
    }

    return $this;
}

public function unless($value, $callback, $default = null)
{
    if (! $value) {
        return $callback($this, $value) ?: $this;
    } elseif ($default) {
        return $default($this, $value) ?: $this;
    }

    return $this;
}

public function tap($callback)
{
    return $this->when(true, $callback);
}
```

## Aggregate 查询

聚合方法也是 sql 的重要组成部分，`laravel` 提供 `count`、`max`、`min`、`avg`、`sum`、`exist` 等聚合方法：

```php
DB::table('users')->count();//select count(*) as aggregate from "users"

DB::table('users')->max('id');//select max("id") as aggregate from "users"

DB::table('users')->min('id');//select min("id") as aggregate from "users"

DB::table('users')->sum('id');//select sum("id") as aggregate from "users"

DB::table('users')->exists();//select exists(select * from "users") as "exists"

```
这些聚合函数实际上都是调用 `aggregate` 函数：
 
```php
public function count($columns = '*')
{
    return (int) $this->aggregate(__FUNCTION__, Arr::wrap($columns));
}

public function aggregate($function, $columns = ['*'])
{
    $results = $this->cloneWithout(['columns'])
                    ->cloneWithoutBindings(['select'])
                    ->setAggregate($function, $columns)
                    ->get($columns);

    if (! $results->isEmpty()) {
        return array_change_key_case((array) $results[0])['aggregate'];
    }
}
```
可以看出来，`aggregate` 函数复制了一份 `queryBuilder`，只是缺少了 `select`、`bingding` 成员变量:

```php
public function cloneWithout(array $properties)
{
    return tap(clone $this, function ($clone) use ($properties) {
        foreach ($properties as $property) {
            $clone->{$property} = null;
        }
    });
}

public function cloneWithoutBindings(array $except)
{
    return tap(clone $this, function ($clone) use ($except) {
        foreach ($except as $type) {
            $clone->bindings[$type] = [];
        }
    });
}

protected function setAggregate($function, $columns)
{
    $this->aggregate = compact('function', 'columns');

    if (empty($this->groups)) {
        $this->orders = null;

        $this->bindings['order'] = [];
    }

    return $this;
}
```
`exist` 聚合函数和其他不一样，它的流程与 `whereExist` 大致相同：

```php
public function exists()
{
    $results = $this->connection->select(
        $this->grammar->compileExists($this), $this->getBindings(), ! $this->useWritePdo
    );

    if (isset($results[0])) {
        $results = (array) $results[0];

        return (bool) $results['exists'];
    }

    return false;
}

```

### grammer——compileAggregate

`laravel` 的聚合函数具有独占性，也就是说调用聚合函数后，不能再 `select` 其他的列:

```php

protected function compileAggregate(Builder $query, $aggregate)
{
    $column = $this->columnize($aggregate['columns']);

    if ($query->distinct && $column !== '*') {
        $column = 'distinct '.$column;
    }

    return 'select '.$aggregate['function'].'('.$column.') as aggregate';
}

protected function compileColumns(Builder $query, $columns)
{
    if (! is_null($query->aggregate)) {
        return;
    }

    $select = $query->distinct ? 'select distinct ' : 'select ';

    return $select.$this->columnize($columns);
}
```

可以看到，如果存在聚合函数，那么编译 `select` 的 `compileColumns` 函数将不会运行。

## first / find / value / pluck / implode

从数据库中取出后数据，我们可以使用 `laravel` 提供给我们的一些函数进行包装处理。

`first` 函数可以让我们只查询第一条：

```php
DB::table('users')->where('id', '=', 1)->first();
```

`find` 函数，可以利用数据库表的主键来查询第一条：

```
DB::table('users')->find(1);
```

`pluck` 函数可以取查询记录的某一列：

```php
DB::table('users')->where('id', '=', 1)->pluck('foo');/['bar', 'baz']
```

`pluck` 函数取查询记录的某一列的同时，还可以设置列名的 `key`： 

```php
DB::table('users')->where('id', '=', 1)->pluck('foo', 'id');//[1 => 'bar', 10 => 'baz']
```

`value` 函数可以取第一条数据的某一列：

```php
DB::table('users')->where('id', '=', 1)->value('foo');//bar

```

`implode` 函数可以将多条数据的某一列拼成字符串：

```php
DB::table('users')->where('id', '=', 1)->implode('foo', ',');//'bar,baz'
```

### first 函数

`find` 函数，使用了 `limit 1` 的 `sql` 语句：

```php
public function first($columns = ['*'])
{
    return $this->take(1)->get($columns)->first();
}

```

### find 函数

`find` 函数实际利用主键调用 `first` 函数：

```php
public function find($id, $columns = ['*'])
{
    return $this->where('id', '=', $id)->first($columns);
}
```

### pluck 函数

`pluck` 函数主要对得到的数据调用 `pluck` 函数：

```php
public function pluck($column, $key = null)
{
    $results = $this->get(is_null($key) ? [$column] : [$column, $key]);

    return $results->pluck(
        $this->stripTableForPluck($column),
        $this->stripTableForPluck($key)
    );
}

```

### implod 函数

`implod` 函数对一维数组调用 `implod` 函数：

```php
public function implode($column, $glue = '')
{
    return $this->pluck($column)->implode($glue);
}

```

## chunk 语句

如果你需要操作数千条数据库记录，可以考虑使用 chunk 方法。这个方法每次只取出一小块结果，并会将每个块传递给一个闭包处理。

```php
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
    foreach ($users as $user) {
        //
    }
});

```
你可以从 闭包 中返回 false，以停止对后续分块的处理：

```php
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
    // Process the records...
    if (...) {
        return false;
    } 
});

```

如果不想按照主键 id 来进行分块，我们还可以自定义分块主键：

```php
DB::table('users')->orderBy('id')->chunkById(100, function ($users) {
    foreach ($users as $user) {
        //
    }
}, 'someIdField');

```

### chunk 函数

`chunk` 函数的实现实际上是 `forPage` 函数，当从数据库获得数据后，先判断是否拿到了数据，如果拿到了就会继续执行闭包函数，否则就会中断程序。执行闭包函数后，需要判断返回状态。若取出的数据小于分块的条数，说明数据已经全部获取完毕，结束程序。

```php
public function chunk($count, callable $callback)
{
    $this->enforceOrderBy();

    $page = 1;

    do {
        $results = $this->forPage($page, $count)->get();

        $countResults = $results->count();

        if ($countResults == 0) {
            break;
        }

        if ($callback($results, $page) === false) {
            return false;
        }

        unset($results);

        $page++;
    } while ($countResults == $count);

    return true;
}

protected function enforceOrderBy()
{
    if (empty($this->query->orders) && empty($this->query->unionOrders)) {
        $this->orderBy($this->model->getQualifiedKeyName(), 'asc');
    }
}
```
`enforceOrderBy` 函数是用于数据按照主键的大小进行排序。

### chunkById 函数

chunkById 函数与 chunk 函数唯一不同的是 `forPage` 函数被换成了 `forPageAfterId` 函数，目的是替换主键：
 
```php
public function chunkById($count, callable $callback, $column = 'id', $alias = null)
{
    $alias = $alias ?: $column;

    $lastId = 0;

    do {
        $clone = clone $this;

        $results = $clone->forPageAfterId($count, $lastId, $column)->get();

        $countResults = $results->count();

        if ($countResults == 0) {
            break;
        }

        if ($callback($results) === false) {
            return false;
        }

        $lastId = $results->last()->{$alias};

        unset($results);
    } while ($countResults == $count);

    return true;
}

```

`forPageAfterId` 函数实际上是把 `offset` 函数删除，并按照自定义的列来排序，每次获取最后一条数据的自定义列的数值，利用 `where` 条件不断获取下一部分分块数据：

```php
public function forPageAfterId($perPage = 15, $lastId = 0, $column = 'id')
{
    $this->orders = $this->removeExistingOrdersFor($column);

    return $this->where($column, '>', $lastId)
                ->orderBy($column, 'asc')
                ->take($perPage);
}

protected function removeExistingOrdersFor($column)
{
    return Collection::make($this->orders)
                ->reject(function ($order) use ($column) {
                    return isset($order['column'])
                           ? $order['column'] === $column : false;
                })->values()->all();
}
```