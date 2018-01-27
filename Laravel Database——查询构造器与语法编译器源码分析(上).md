title: Laravel Database——查询构造器与语法编译器源码分析(上)

tags:
  - php
  - laravel
  - database
  - 源码
categories:
  - php
  - database
  - laravel
date: 2017-09-19 23:46:36
---

---

## 前言

在前两个文章中，我们分析了数据库的连接启动与数据库底层 `CRUD` 的原理，底层数据库服务支持原生 `sql` 的运行。本文以 `mysql` 为例，向大家讲述支持 `Fluent` 的查询构造器 `query` 与语法编译器 `grammer` 的原理。

## DB::table 与 查询构造器

若是不想使用原生的 `sql` 语句，我们可以使用 `DB::table` 语句，该语句会返回一个 `query` 对象：

```php
public function table($table)
{
    return $this->query()->from($table);
}

public function query()
{
    return new QueryBuilder(
        $this, $this->getQueryGrammar(), $this->getPostProcessor()
    );
}
```
我们可以看到，`query` 会有两个成员，`queryGrammar` 与 `postProcessor`。`queryGrammar` 负责对 `QueryBuilcder` 的结果进行 `sql` 语言的转化，`postProcessor` 负责查询结果的后处理。

之所以 `laravel` 推荐我们使用查询构造器，而不是原生的 `sql`，原因在于可以避免 `sql` 注入漏洞。当然，我们也可以在使用 `DB::select()` 函数中手动写 `bindings` 的值，但是这样的话，我们写 `sql` 的语句是就必须是这样：

```php
DB::select('select * from table where col=?',[1]);
```

必然会带来很多不便。

有了查询构造器，我们就可以写出 `fluent` 类型的语句：

```php
DB::table('table')->select('*')->where('col', 1);

```

是不是很方便？

## CRUD 与语法编译器

相应于 `connection` 对象的 `CRUD`，语法编译器有 `compileInsert`、`compileSelect`、`compileUpdate`、`compileDelete`。其中最重要的是 `compileSelect`，因为它不仅负责了 `select` 语句的语法编译，还负责聚合语句 `aggregate`、`from` 语句、`join` 连接语句、`wheres` 条件语句、`groups` 分组语句、`havings` 条件语句、`orders` 排序语句、`limit` 语句、`offset` 语句、`unions` 联合语句、`lock` 语句：

```php
protected $selectComponents = [
    'aggregate',
    'columns',
    'from',
    'joins',
    'wheres',
    'groups',
    'havings',
    'orders',
    'limit',
    'offset',
    'unions',
    'lock',
];

public function compileSelect(Builder $query)
{
   
    $original = $query->columns;

    if (is_null($query->columns)) {
        $query->columns = ['*'];
    }

    $sql = trim($this->concatenate(
        $this->compileComponents($query))
    );

    $query->columns = $original;

    return $sql;
}

protected function compileComponents(Builder $query)
{
    $sql = [];

    foreach ($this->selectComponents as $component) {
        if (! is_null($query->$component)) {
            $method = 'compile'.ucfirst($component);

            $sql[$component] = $this->$method($query, $query->$component);
        }
    }

    return $sql;
}
```

可以看出来，语法编译器会将上述所有的语句放入 `$sql[]` 成员中，然后通过 `concatenate` 函数组装成 `sql` 语句：

```php
protected function concatenate($segments)
{
    return implode(' ', array_filter($segments, function ($value) {
        return (string) $value !== '';
    }));
}

```

### wrap 函数

若想要了解语法编译器，我们就必须先要了解 `grammer` 中一个重要的函数 `wrap`，这个函数专门对表名与列名进行处理，

```php
public function wrap($value, $prefixAlias = false)
{
    if ($this->isExpression($value)) {
        return $this->getValue($value);
    }

    if (strpos(strtolower($value), ' as ') !== false) {
        return $this->wrapAliasedValue($value, $prefixAlias);
    }

    return $this->wrapSegments(explode('.', $value));
}
```

处理的流程：

- 若是 `Expression` 对象，利用函数 `getValue` 直接取出对象值，不对其进行任何处理，用于处理原生 `sql`。`expression` 对象的作用是保护原始参数，避免框架解析的一种方式。也就是说，当我们用了 `expression` 来包装参数的话，`laravel` 将不会对其进行任何处理，包括库名解析、表名前缀、别名等。
- 若表名/列名存在 `as`，则利用函数 `wrapAliasedValue` 为表名设置别名。
- 若表名/列名含有 `.`，则会被分解为 `库名/表名`，或者 `表名/列名`，并调用函数 `wrapSegments`。

### wrapAliasedValue 函数

`wrapAliasedValue` 函数用于处理别名：

```php
protected function wrapAliasedValue($value, $prefixAlias = false)
{
    $segments = preg_split('/\s+as\s+/i', $value);

    if ($prefixAlias) {
        $segments[1] = $this->tablePrefix.$segments[1];
    }

    return $this->wrap(
        $segments[0]).' as '.$this->wrapValue($segments[1]
    );
}
```
可以看到，首先程序会根据 `as` 将字符串分为两部分，`as` 前的部分递归调用 `wrap` 函数，`as` 后的部分调用 `wrapValue` 函数.

### wrapValue 函数

`wrapValue` 函数用来处理添加符号 `"`，例如 `table`，会被这个函数变为 `"table"`。需要注意的是 `table1"table2` 这种情况，假如我们的数据库中存在一个表，名字就叫做: `table1"table2`，我们在数据库查询的时候，必须将表名转化为 `"table1""table2"`，只有这样，数据库才会有效地转化表名为 `"table1"table2"`，否则数据库就会报告错误：找不到表。

```php
protected function wrapValue($value)
{
    if ($value !== '*') {
        return '"'.str_replace('"', '""', $value).'"';
    }

    return $value;
}
```

### wrapSegments 函数

`wrapSegments` 函数会判断当前参数，如果是 `table.column`，会将前一部分 `table` 调用 `wrapTable`, `column` 调用 `wrapValue`，最后生成 `“table”."column"`。

```php
protected function wrapSegments($segments)
{
    return collect($segments)->map(function ($segment, $key) use ($segments) {
        return $key == 0 && count($segments) > 1
                        ? $this->wrapTable($segment)
                        : $this->wrapValue($segment);
    })->implode('.');
}
```

### wrapTable 函数

`wrapTable` 函数用于为数据表添加表前缀：

```
public function wrapTable($table)
{
    if (! $this->isExpression($table)) {
        return $this->wrap($this->tablePrefix.$table, true);
    }

    return $this->getValue($table);
}
```

`wrap` 整体流程图如下：

![Markdown](http://owql68l6p.bkt.clouddn.com/17-9-23/27965477.jpg)
 
## from 语句

我们看到 `DB::table` 实际上是调用了查询构造器的 `from` 函数。接下来我们就看看，当我们写下了

```php
DB::table('table')->get()
```
时发生了什么。

```php
public function from($table)
{
    $this->from = $table;

    return $this;
}
```
我们看到，`from` 函数极其简单，我们接下来看 `get`：

```php
public function get($columns = ['*'])
{
    $original = $this->columns;

    if (is_null($original)) {
        $this->columns = $columns;
    }

    $results = $this->processor->processSelect($this, $this->runSelect());

    $this->columns = $original;

    return collect($results);
}
```
`laravel` 的查询构造器是懒加载的，只有调用了 `get` 函数才会真正的调用语法编译器，采用调用底层 `connection` 对象进行数据库查询：

```php
protected function runSelect()
{
    return $this->connection->select(
        $this->toSql(), $this->getBindings(), ! $this->useWritePdo
    );
}

public function toSql()
{
    return $this->grammar->compileSelect($this);
}
```
### compileFrom 函数

首先我们先看看流程图：

![Markdown](http://owql68l6p.bkt.clouddn.com/17-9-23/10116808.jpg)

语法编译器 `grammer` 对于 `from` 语句的处理由函数 `compileFrom` 负责：

```php
protected function compileFrom(Builder $query, $table)
{
    return 'from '.$this->wrapTable($table);
}
```
从流程图可以看出，具体流程与 `wrap` 类似。

我们调用 `from` 时，可以传递两种参数，一种是字符串，另一种是 `expression` 对象：

```php
DB::table('table');
DB::table(new Expression('table'));
```

- 传递 `expression` 对象

当我们传递 `expression` 对象的时候，`grammer` 就会调用 `getValue` 取出原生 `sql` 语句。

- 传递字符串

当我们向 `from` 传递普通的字符串时，`laravel` 就会对字符串调用 `wrap` 函数进行处理，处理流程上一个小节已经说明：

- 为表名加上前缀 `$this->tablePrefix`
- 若字符串存在 `as`，则为表名设置别名。
- 若字符串含有 `.`，则会被分解为 `库名` 与 `表名`，并进行分别调用 `wrapTable` 函数与 `wrapValue` 进行处理。
- 为表名前后添加 `"`，例如 `t1.t2` 会被转化为 `"t1"."t2"`（不同的数据库添加的字符不同，mysql 就不是 `"`）

`laravel` 对 `from` 处理流程存在一些问题，表名前缀设置功能与数据库名功能公用存在问题，相关 `issue` 地址是：[[Bug] Table prefix added to database name when using database.table](https://github.com/laravel/framework/issues/21305)，有任何兴趣的同学可以在这个 `issue` 里面讨论，或者直接向作者提 `PR`。若作者对此部分有任何修改，我会同步修改这篇文章。

## Select 语句

本小节会介绍 `select` 语句：

```php
public function select($columns = ['*'])
{
    $this->columns = is_array($columns) ? $columns : func_get_args();

    return $this;
}
```
`queryBuilder` 的 `select` 语句很简单，我们不多讨论.

`selectRaw` ：

```php
public function selectRaw($expression, array $bindings = [])
{
    $this->addSelect(new Expression($expression));

    if ($bindings) {
        $this->addBinding($bindings, 'select');
    }

    return $this;
}

public function addSelect($column)
{
    $column = is_array($column) ? $column : func_get_args();

    $this->columns = array_merge((array) $this->columns, $column);

    return $this;
}
```

可以看到， `selectRaw` 就是将 `Expression` 对象赋值到 `columns` 中，我们在前面说到，框架不会对 `Expression` 进行任何处理（更准确的说是 `wrap` 函数），这样就保证了原生语句的执行。

我们接着看 `grammer` .

### compileColumns 函数

```php
protected function compileColumns(Builder $query, $columns)
{
    if (! is_null($query->aggregate)) {
        return;
    }

    $select = $query->distinct ? 'select distinct ' : 'select ';

    return $select.$this->columnize($columns);
}

public function columnize(array $columns)
{
    return implode(', ', array_map([$this, 'wrap'], $columns));
}
```
可以看到，`grammer` 对 `select` 的语法编译调用 `wrap` 函数对每个 `select` 的字段进行处理，处理过程在上面详解过，在此不再赘述。

## selectSub 语句

所谓的 `select` 子查询，就是查询的字段来源于其他数据表。对于这种查询，可以分成两部来理解，首先忽略整个select子查询，查出第一个表中的数据，然后根据第一个表的数据执行子查询，

`laravel` 的 `selectSub` 支持闭包函数、`queryBuild` 对象或者原生 `sql` 语句，以下是单元测试样例：

```php
$query = DB::table('one')->select(['foo', 'bar'])->where('key', '=', 'val');

$query->selectSub(function ($query) {
        $query->from('two')->select('baz')->where('subkey', '=', 'subval');
    }, 'sub');
```
另一种写法：

```php
$query = DB::table('one')->select(['foo', 'bar'])->where('key', '=', 'val');
$query_sub = DB::table('one')->select('baz')->where('subkey', '=', 'subval');

$query->selectSub($query_sub, 'sub');
```
生成的 `sql`：

```
select "foo", "bar", (select "baz" from "two" where "subkey" = 'subval') as "sub" from "one" where "key" = 'val'
```   

`selectSub` 语句的实现比较简单：

```php
public function selectSub($query, $as)
{
    if ($query instanceof Closure) {
        $callback = $query;

        $callback($query = $this->forSubQuery());
    }

    list($query, $bindings) = $this->parseSubSelect($query);

    return $this->selectRaw(
        '('.$query.') as '.$this->grammar->wrap($as), $bindings
    );
}

protected function parseSubSelect($query)
{
    if ($query instanceof self) {
        $query->columns = [$query->columns[0]];

        return [$query->toSql(), $query->getBindings()];
    } elseif (is_string($query)) {
        return [$query, []];
    } else {
        throw new InvalidArgumentException;
    }
}
```
可以看到，如果 `selectSub` 的参数是闭包函数，那么就会先执行闭包函数，闭包函数将会为 `query` 根据查询语句更新对象。

`parseSubSelect` 函数为子查询解析 `sql` 语句与 `binding` 变量。

## where 语句总结

在 `laravel` 文档中，`queryBuild` 的用法很详尽，但是为了更好的理解源码，我们在这里再次大概的总结一下：

### 基础用法

```php
users = DB::table('users')->where('votes', '=', 100)->get();
$users = DB::table('users')->where('votes', 100)->get();
```
这两个是等价的写法。

### `where` 数组

```php
$users = DB::table('users')->where(
    ['status' => 1, 'subscribed' => 1],
)->get();

$users = DB::table('users')->where([
    ['status', '1'],
    ['subscribed', '1'],
])->get();

$users = DB::table('users')->where([
    ['status', '1'],
    ['subscribed', '<>', '1'],
])->get();

```
### `where` 查询组

```php
DB::table('users')
    ->where('name', '=', 'John')
    ->orWhere(function ($query) {
        $query->where('votes', '>', 100)
              ->where('title', '<>', 'Admin');
    })
    ->get();
```
这一句的 `sql` 语句是 

```php
select * from users where name = 'John' or (votes > 100 and title <> 'Admin')
```

### `where` 子查询

```php
public function testFullSubSelects()
{
    $builder = $this->getBuilder();
    DB::table('users')
        ->Where('id', '=', function ($q) {
        $q->select(new Raw('max(id)'))->from('users')->where('email', '=', 'bar');
    });
}
```
这一句的 `sql` 语句是 

```php
select * from "users" where "email" = foo or "id" = (select max(id) from "users" where "email" = bar`
```

### orWhere

```php
$users = DB::table('users')
            ->where('votes', '>', 100)
            ->orWhere('name', 'John')
            ->get();
```

### whereDate / whereMonth / whereDay / whereYear / whereTime

```php
$users = DB::table('users')
	        ->whereDate('created_at', '2016-12-31')
	        ->get();
	        
$users = DB::table('users')
            ->whereMonth('created_at', '12')
            ->get();	        

$users = DB::table('users')
            ->whereDay('created_at', '31')
            ->get();
            
$users = DB::table('users')
            ->whereYear('created_at', '2016')
            ->get();
            
$users = DB::table('users')
            ->whereTime('created_at', '>=', '22:00')
            ->get();
```
### whereBetween / whereNotBetween

```php
$users = DB::table('users')
            ->whereBetween('votes', [1, 100])->get();
            
$users = DB::table('users')
            ->whereNotBetween('votes', [1, 100])
            ->get();
```

### whereRaw / orWhereRaw 

```php
$users = DB::table('users')
            ->whereRaw('id = ? or email = ?', [1, 'foo'])
            ->get();
            
$users = DB::table('users')
            ->orWhereRaw('id = ? or email = ?', [1, 'foo'])
            ->get();
```

### whereIn / whereNotIn / orWhereIn / orWhereNotIn

```php
$users = DB::table('users')
            ->whereIn('id', [1, 2, 3])
            ->get();

$users = DB::table('users')
            ->whereNotIn('id', [1, 2, 3])
            ->get();

$users = DB::table('users')            
				->whereIn('id', function ($q) {
        			$q->select('id')->from('users')->where('age', '>', 25)->take(3);
    			});       
    			
$users = DB::table('users')            
				->whereNotIn('id', function ($q) {
        			$q->select('id')->from('users')->where('age', '>', 25)->take(3);
    			});  
    			
$query = DB::table('users')->select('id')->where('age', '>', 25)->take(3);
$users = DB::table('users')->whereIn('id', $query);   
$users = DB::table('users')->whereNotIn('id', $query);
			     
```
有意思的是，当我们在 `whereIn / whereNotIn / orWhereIn / orWhereNotIn` 中传入空数组的时候：
   
```php            
$users = DB::table('users')
            ->whereIn('id', [])
            ->get();

$users = DB::table('users')
            ->orWhereIn('id', [])
            ->get();
```
这个时候，框架自动会生成如下的 `sql`：

```php
select * from "users" where 0 = 1;

select * from "users" where "id" = ? or 0 = 1
```

### whereColumn

```php
$users = DB::table('users')
            ->whereColumn('first_name', 'last_name')
            ->get();

$users = DB::table('users')
            ->whereColumn('updated_at', '>', 'created_at')
            ->get();

$users = DB::table('users')
            ->whereColumn([
                ['first_name', '=', 'last_name'],
                ['updated_at', '>', 'created_at']
            ])->get();            
```

### whereNull / whereNotNull / orWhereNull / orWhereNotNull

```php
$users = DB::table('users')
            ->whereNull('updated_at')
            ->get();

$users = DB::table('users')
            ->whereNotNull('updated_at')
            ->get();
            
$users = DB::table('users')
            ->orWhereNull('updated_at')
            ->get();
            
$users = DB::table('users')
            ->orWhereNotNull('updated_at')
            ->get();                        
```

### whereExists / whereNotExists / orWhereExists / orWhereNotExists

```php
DB::table('users')
    ->whereExists(function ($query) {
        $query->select(DB::raw(1))
              ->from('orders')
              ->whereRaw('orders.user_id = users.id');
    })
    ->get(); 
// select * from users where exists ( select 1 from orders where orders.user_id = users.id)

DB::table('users')
    ->whereNotExists(function ($query) {
        $query->select(DB::raw(1))
              ->from('orders')
              ->whereRaw('orders.user_id = users.id');
    })
    ->get();
// select * from users where not exists ( select 1 from orders where orders.user_id = users.id)
    
DB::table('users')
    ->orWhereExists(function ($query) {
        $query->select(DB::raw(1))
              ->from('orders')
              ->whereRaw('orders.user_id = users.id');
    })
    ->get();
// select * from users or exists ( select 1 from orders where orders.user_id = users.id)
    
DB::table('users')
    ->orWhereNotExists(function ($query) {
        $query->select(DB::raw(1))
              ->from('orders')
              ->whereRaw('orders.user_id = users.id');
    })
    ->get();  
// select * from users or not exists ( select 1 from orders where orders.user_id = users.id)             
```

## where 函数

我们首先先看看源码：

```php
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    if (is_array($column)) {
        return $this->addArrayOfWheres($column, $boolean);
    }

    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    if ($column instanceof Closure) {
        return $this->whereNested($column, $boolean);
    }

    if ($this->invalidOperator($operator)) {
        list($value, $operator) = [$operator, '='];
    }

    if ($value instanceof Closure) {
        return $this->whereSub($column, $operator, $value, $boolean);
    }

    if (is_null($value)) {
        return $this->whereNull($column, $boolean, $operator !== '=');
    }

    if (Str::contains($column, '->') && is_bool($value)) {
        $value = new Expression($value ? 'true' : 'false');
    }

    $type = 'Basic';

    $this->wheres[] = compact(
        'type', 'column', 'operator', 'value', 'boolean'
    );

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'where');
    }

    return $this;
}
```
可以看到，为了支持框架的多种 `where` 形式，`where` 的代码中写了很多的条件语句。我们接下来一个个分析。

### grammer——compileWheres 函数

在此之前，我们先看看语法编译器对 `where` 查询的处理：

```php
protected function compileWheres(Builder $query)
{
    if (is_null($query->wheres)) {
        return '';
    }

    if (count($sql = $this->compileWheresToArray($query)) > 0) {
        return $this->concatenateWhereClauses($query, $sql);
    }

    return '';
}
```

`compileWheres` 函数负责所有 `where` 查询条件的语法编译工作，`compileWheresToArray` 函数负责循环编译查询条件，`concatenateWhereClauses` 函数负责将多个查询条件合并。

```php
protected function compileWheresToArray($query)
{
    return collect($query->wheres)->map(function ($where) use ($query) {
        return $where['boolean'].' '.$this->{"where{$where['type']}"}($query, $where);
    })->all();
}

protected function concatenateWhereClauses($query, $sql)
{
    $conjunction = $query instanceof JoinClause ? 'on' : 'where';

    return $conjunction.' '.$this->removeLeadingBoolean(implode(' ', $sql));
}
```

`compileWheresToArray` 函数负责把 `$query->wheres` 中多个 `where` 条件循环起来：

- `$where['boolean']` 是多个查询条件的连接，`and` 或者 `or`，一般 `where` 条件默认为 `and`,各种 `orWhere` 的连接是 `or`
- `where{$where['type']}` 是查询的类型，`laravel` 把查询条件分为以下几类：`base`、`raw`、`in`、`notIn`、`inSub`、`notInSub`、`null`、`notNull`、`between`、`column`、`nested`、`sub`、`exist`、`notExist`。每种类型的查询条件都有对应的 `grammer` 方法

`concatenateWhereClauses` 函数负责连接所有的搜索条件，由于 `join` 的连接条件也会调用 `compileWheres` 函数，所以会有判断是否是真正的 `where` 查询，

### where 数组

如果 `column` 是数组的话，就会调用：

```php
protected function addArrayOfWheres($column, $boolean, $method = 'where')
{
    return $this->whereNested(function ($query) use ($column, $method, $boolean) {
        foreach ($column as $key => $value) {
            if (is_numeric($key) && is_array($value)) {
                $query->{$method}(...array_values($value));
            } else {
                $query->$method($key, '=', $value, $boolean);
            }
        }
    }, $boolean);
}
```

可以看到，数组分为两类，一种是列名为 `key`，例如 `['foo' => 1, 'bar' => 2]`，这个时候就是调用 `query->where('foo', '=', '1', ‘and’)`。还有一种是 `[['foo','1'],['bar','2']]`，这个时候就会调用 `$query->where(['foo','1'])`。

```php
public function whereNested(Closure $callback, $boolean = 'and')
{
    call_user_func($callback, $query = $this->forNestedWhere());

    return $this->addNestedWhereQuery($query, $boolean);
}

public function addNestedWhereQuery($query, $boolean = 'and')
{
    if (count($query->wheres)) {
        $type = 'Nested';

        $this->wheres[] = compact('type', 'query', 'boolean');

        $this->addBinding($query->getBindings(), 'where');
    }

    return $this;
}
```

#### grammer——whereNested

语法编译器中负责查询组的函数是 `whereNested`，它会取出 `where` 中的 `query`，递归调用 `compileWheres` 函数

```php
protected function whereNested(Builder $query, $where)
{
    $offset = $query instanceof JoinClause ? 3 : 6;

    return '('.substr($this->compileWheres($where['query']), $offset).')';
}
```

由于 `compileWheres` 会返回 `where ...` 或者 `on ...` 等开头的 `sql` 语句，所以我们需要把返回结果截取前3个字符或6个字符。
 
### where 查询组

若查询条件是一个闭包函数，也就是第一个参数 `column` 是个闭包函数，那么就要调用 `whereNested` 函数，过程和上述过程一致。

### whereSub 子查询

如果第二个参数或者第三个参数是一个闭包函数的话，就是 `where` 子查询语句，这时需要调用 `whereSub` 函数：

```php
protected function whereSub($column, $operator, Closure $callback, $boolean)
{
    $type = 'Sub';

    call_user_func($callback, $query = $this->forSubQuery());

    $this->wheres[] = compact(
        'type', 'column', 'operator', 'query', 'boolean'
    );

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

#### grammer——whereSub

`grammer` 中负责子查询的是 `whereSub` 函数： 

```php
protected function whereSub(Builder $query, $where)
{
    $select = $this->compileSelect($where['query']);

    return $this->wrap($where['column']).' '.$where['operator']." ($select)";
}
```
因为子查询中可以存在 `select` 、`where` 、`join` 等一切 `sql` 语句，所以递归的是 `compileSelect` 这个大的函数，而不是仅仅 `compileWheres`。

### whereNull 语句

`whereNull` 函数也很简单：

```php
public function whereNull($column, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotNull' : 'Null';

    $this->wheres[] = compact('type', 'column', 'boolean');

    return $this;
}
```

#### grammer——whereNull 函数

```php
protected function whereNull(Builder $query, $where)
{
    return $this->wrap($where['column']).' is null';
}
```

### whereBasic 语句

如果上述情况都不符合，那么就是最基础的 `where` 语句，类型是 `basic`.

```php
$type = 'Basic';

$this->wheres[] = compact(
    'type', 'column', 'operator', 'value', 'boolean'
);

if (! $value instanceof Expression) {
    $this->addBinding($value, 'where');
}
```

#### grammer——whereBasic 函数

`grammer` 中最基础的 `where` 语句由 `wherebasic` 函数负责：

```php
protected function whereBasic(Builder $query, $where)
{
    $value = $this->parameter($where['value']);

    return $this->wrap($where['column']).' '.$where['operator'].' '.$value;
}

public function parameter($value)
{
    return $this->isExpression($value) ? $this->getValue($value) : '?';
}
```

`wherebasic` 函数对参数进行了替换，利用 `?` 来替换真正的值。

## orWhere 语句

`orWhere` 函数只是在 `where` 函数的基础上固定了最后一个参数：

```php
public function orWhere($column, $operator = null, $value = null)
{
    return $this->where($column, $operator, $value, 'or');
}
```

## whereColumn 语句

`whereColumn` 函数是简化版的 `where` 函数，只是 `where` 类型不是 `basic`，而是 `column`：

```php
public function whereColumn($first, $operator = null, $second = null, $boolean = 'and')
{
    if (is_array($first)) {
        return $this->addArrayOfWheres($first, $boolean, 'whereColumn');
    }

    if ($this->invalidOperator($operator)) {
        list($second, $operator) = [$operator, '='];
    }

    $type = 'Column';

    $this->wheres[] = compact(
        'type', 'first', 'operator', 'second', 'boolean'
    );

    return $this;
}
```

### grammer——whereColumn

可以看到 `whereColumn` 与 `whereBasic` 的区别是对 `value` 的不同处理，`whereBasic` 实际上是将其看作值，需要用 `?` 来替换，参数加载到 `binding` 中去的。而 `whereColumn` 是将 `second` 当做列名来处理，是需要经过表名、别名等处理的：

```php
protected function whereColumn(Builder $query, $where)
{
    return $this->wrap($where['first']).' '.$where['operator'].' '.$this->wrap($where['second']);
}
```

## whereIn 语句

```php
public function whereIn($column, $values, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotIn' : 'In';

    if ($values instanceof EloquentBuilder) {
        $values = $values->getQuery();
    }

    if ($values instanceof self) {
        return $this->whereInExistingQuery(
            $column, $values, $boolean, $not
        );
    }

    if ($values instanceof Closure) {
        return $this->whereInSub($column, $values, $boolean, $not);
    }

    if ($values instanceof Arrayable) {
        $values = $values->toArray();
    }

    $this->wheres[] = compact('type', 'column', 'values', 'boolean');

    foreach ($values as $value) {
        if (! $value instanceof Expression) {
            $this->addBinding($value, 'where');
        }
    }

    return $this;
}
```

可以看出来，`whereIn` 支持四种参数: `EloquentBuilder`、`queryBuilder`、`Closure`、`Arrayable`。

### whereInExistingQuery 函数

当 `whereIn` 第二个参数是 `queryBuild` 时，就会调用 `whereInExistingQuery` 函数：

```php
protected function whereInExistingQuery($column, $query, $boolean, $not)
{
    $type = $not ? 'NotInSub' : 'InSub';

    $this->wheres[] = compact('type', 'column', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```
可以看出，这个函数添加了一个类型为 `InSub` 或 `NotInSub` 类型的 `where`，我们接着在语法编译器来看：

#### grammer——whereInSub / whereNotInSub

`whereInSub` / `whereNotInSub` 与 `whereSub` 类似，只是 `operator` 被固定成 `in` / `not in` 而已：

```php
protected function whereInSub(Builder $query, $where)
{
    return $this->wrap($where['column']).' in ('.$this->compileSelect($where['query']).')';
}

protected function whereNotInSub(Builder $query, $where)
{
    return $this->wrap($where['column']).' not in ('.$this->compileSelect($where['query']).')';
}
```

### whereInSub 函数

当 `whereIn` 第二个参数是闭包函数的时候，就会调用 `whereInSub` 函数：

```php
protected function whereInSub($column, Closure $callback, $boolean, $not)
{
    $type = $not ? 'NotInSub' : 'InSub';

    call_user_func($callback, $query = $this->forSubQuery());

    $this->wheres[] = compact('type', 'column', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

可以看出来，除了闭包函数需要执行获得 `query` 对象之外，`whereInSub` 函数与 `whereInExistingQuery` 函数一致。

### In / NotIn 类型 where

如果参数传递了数组，那么就会创建 `In` 或者 `NotIn` 类型的 `where`:

```php
$this->wheres[] = compact('type', 'column', 'values', 'boolean');

foreach ($values as $value) {
    if (! $value instanceof Expression) {
        $this->addBinding($value, 'where');
    }
}
```

#### grammer——whereIn / whereNotIn

`whereIn` 或者 `whereNotIn` 与 `whereBasic` 函数基本一致，只是参数值需要循环调用 `parameter` 函数：

```php
protected function whereIn(Builder $query, $where)
{
    if (! empty($where['values'])) {
        return $this->wrap($where['column']).' in ('.$this->parameterize($where['values']).')';
    }

    return '0 = 1';
}

protected function whereNotIn(Builder $query, $where)
{
    if (! empty($where['values'])) {
        return $this->wrap($where['column']).' not in ('.$this->parameterize($where['values']).')';
    }

    return '1 = 1';
}

public function parameterize(array $values)
{
    return implode(', ', array_map([$this, 'parameter'], $values));
}
```

## whereBetween 语句

类似的 `whereBetween` 创建了新的 `between` 类型的 `where`：

```php
public function whereBetween($column, array $values, $boolean = 'and', $not = false)
{
    $type = 'between';

    $this->wheres[] = compact('column', 'type', 'boolean', 'not');

    $this->addBinding($values, 'where');

    return $this;
}
```

### grammer——whereBetween

有意思的是，`whereBetween` 不支持原生的参数值，也就是不支持 `expression` 对象，所以直接用 `?` 来代替参数值：

```php
protected function whereBetween(Builder $query, $where)
{
    $between = $where['not'] ? 'not between' : 'between';

    return $this->wrap($where['column']).' '.$between.' ? and ?';
}
```

## whereExist 语句

`whereExist` 只支持闭包函数作为子查询语句，和之前一样，创建了 `Exist` 或者 `NotExist` 类型的 `where`

```php
public function whereExists(Closure $callback, $boolean = 'and', $not = false)
{
    $query = $this->forSubQuery();

    call_user_func($callback, $query);

    return $this->addWhereExistsQuery($query, $boolean, $not);
}

public function addWhereExistsQuery(Builder $query, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotExists' : 'Exists';

    $this->wheres[] = compact('type', 'operator', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}
```

### grammer——whereExists / whereNotExists

可以看到，这部分的语法编译器仍然和 `whereSub` 非常类似：

```php
protected function whereExists(Builder $query, $where)
{
    return 'exists ('.$this->compileSelect($where['query']).')';
}

protected function whereNotExists(Builder $query, $where)
{
    return 'not exists ('.$this->compileSelect($where['query']).')';
}
```

## whereDate / whereMonth / whereDay / whereYear / whereTime

关于时间的查询条件都由函数 `addDateBasedWhere` 负责创建新的类型的 `where`，由于这些函数大致相同，我这里只贴一种：

```php
protected function addDateBasedWhere($type, $column, $operator, $value, $boolean = 'and')
{
    $this->wheres[] = compact('column', 'type', 'boolean', 'operator', 'value');

    $this->addBinding($value, 'where');

    return $this;
}

public function whereDate($column, $operator, $value = null, $boolean = 'and')
{
    list($value, $operator) = $this->prepareValueAndOperator(
        $value, $operator, func_num_args() == 2
    );

    return $this->addDateBasedWhere('Date', $column, $operator, $value, $boolean);
}

...
```
### grammer——dateBasedWhere

时间查询条件都是利用数据库的 `date`、`month`、`day`、`year`、`time` 函数实现的：

```php
protected function whereTime(Builder $query, $where)
{
    return $this->dateBasedWhere('time', $query, $where);
}

protected function dateBasedWhere($type, Builder $query, $where)
{
    $value = $this->parameter($where['value']);

    return $type.'('.$this->wrap($where['column']).') '.$where['operator'].' '.$value;
}
```

## whereRaw 语句

原生的 `sql` 语句及其简单：

```php
public function whereRaw($sql, $bindings = [], $boolean = 'and')
{
    $this->wheres[] = ['type' => 'raw', 'sql' => $sql, 'boolean' => $boolean];

    $this->addBinding((array) $bindings, 'where');

    return $this;
}
```
语法编译器工作：

```php
protected function whereRaw(Builder $query, $where)
{
    return $where['sql'];
}
```













