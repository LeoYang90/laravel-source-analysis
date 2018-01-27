# Laravel Database——Eloquent Model 模型关系加载与查询

## 前言

我们在上一篇文章中介绍了模型关系的定义初始化，我们可以看到，在初始化的过程中 `laravel` 已经为各种关联关系的模型预先插入了初始的 `where` 条件。本文将会进一步介绍如何添加自定义的查询条件，如何加载、预加载关联模型。

## 关联模型的加载

当我们定义关联模型后：

```php
class User extends Model
{
    /**
     * 获得与用户关联的电话记录。
     */
    public function phone()
    {
        $this->hasOne('App\Phone', 'user_id', 'id');
    }
}

```
我们可以像成员变量一样来获取与之关联的模型：

```php
$user = App\User::find(1);

foreach ($user->posts as $post) {
    //
}
```
实际上，模型的属性获取函数的确可以加载关联模型：

```php
public function getAttribute($key)
{
    if (! $key) {
        return;
    }

    ...

    return $this->getRelationValue($key);
}

```

`getRelationValue` 函数专用于加载我们之前定义的关联模型：

```php
public function getRelationValue($key)
{
    if ($this->relationLoaded($key)) {
        return $this->relations[$key];
    }

    if (method_exists($this, $key)) {
        return $this->getRelationshipFromMethod($key);
    }
}

public function relationLoaded($key)
{
    return array_key_exists($key, $this->relations);
}
```

可以看到，关联的加载带有缓存，`laravel` 首先会验证当前关联关系是否已经被加载，如果加载过，那么直接返回缓存结果。

```php
protected function getRelationshipFromMethod($method)
{
    $relation = $this->$method();

    if (! $relation instanceof Relation) {
        throw new LogicException(get_class($this).'::'.$method.' must return a relationship instance.');
    }

    return tap($relation->getResults(), function ($results) use ($method) {
        $this->setRelation($method, $results);
    });
}

```
当我们调用 `$user->posts` 语句的时候，`laravel` 会调用 `posts` 函数，该函数开始定义关联关系，并且返回 `hasOne` 对象，在这里将会调用 `getResults` 函数来加载关联模型：

```php
 public function getResults()
{
    return $this->query->first() ?: $this->getDefaultFor($this->parent);
}
```

`getDefaultFor` 函数用于在未查询到任何关联模型时的情况。我们在定义关联的时候，可以提供默认的方法来控制返回的结果：

```php
public function user()
{
    return $this->belongsTo('App\User')->withDefault();
}

public function user()
{
    return $this->belongsTo('App\User')->withDefault([
        'name' => '游客',
    ]);
}

public function user()
{
    return $this->belongsTo('App\User')->withDefault(function ($user) {
        $user->name = '游客';
    });
}

```

`withDefault` 可以提供空值、数组、闭包函数等等选项，`getDefaultFor` 函数在关联没有查询到结果的时候，按要求返回一个模型：

```php
public function withDefault($callback = true)
{
    $this->withDefault = $callback;

    return $this;
}

protected function getDefaultFor(Model $parent)
{
    if (! $this->withDefault) {
        return;
    }

    $instance = $this->newRelatedInstanceFor($parent);

    if (is_callable($this->withDefault)) {
        return call_user_func($this->withDefault, $instance) ?: $instance;
    }

    if (is_array($this->withDefault)) {
        $instance->forceFill($this->withDefault);
    }

    return $instance;
}
```

获取到关联模型后，就要放入缓存当中，以备后续情况使用：

```php
public function setRelation($relation, $value)
{
    $this->relations[$relation] = $value;

    return $this;
}

```

### 多对多关系的加载

多对多关系的加载与一对多等关系的加载有所不同，原因是不仅要加载 `related` 模型，还要加载中间表模型：

```php
public function getResults()
{
    return $this->get();
}

public function get($columns = ['*'])
{
    $columns = $this->query->getQuery()->columns ? [] : $columns;

    $builder = $this->query->applyScopes();

    $models = $builder->addSelect(
        $this->shouldSelect($columns)
    )->getModels();

    $this->hydratePivotRelation($models);

    if (count($models) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $this->related->newCollection($models);
}
```
`shouldSelect` 函数加载了中间表的字段属性：

```php
protected function shouldSelect(array $columns = ['*'])
{
    if ($columns == ['*']) {
        $columns = [$this->related->getTable().'.*'];
    }

    return array_merge($columns, $this->aliasedPivotColumns());
}

protected function aliasedPivotColumns()
{
    $defaults = [$this->foreignPivotKey, $this->relatedPivotKey];

    return collect(array_merge($defaults, $this->pivotColumns))->map(function ($column) {
        return $this->table.'.'.$column.' as pivot_'.$column;
    })->unique()->all();
}
```
可以看到，这个时候，中间表的属性会被放入 `related` 模型中，并且会被赋予别名前缀 `pivot_`。

接着 `hydratePivotRelation` 会将这些中间表属性加载到中间表模型中：

```php
protected function hydratePivotRelation(array $models)
{
    foreach ($models as $model) {
        $model->setRelation($this->accessor, $this->newExistingPivot(
            $this->migratePivotAttributes($model)
        ));
    }
}

protected function migratePivotAttributes(Model $model)
{
    $values = [];

    foreach ($model->getAttributes() as $key => $value) {
        if (strpos($key, 'pivot_') === 0) {
            $values[substr($key, 6)] = $value;

            unset($model->$key);
        }
    }

    return $values;
}
```

`accessor` 默认值为 `pivot`，我们也可以在定义多对多的时候使用 `as` 函数为它取别名：

```php
return $this->belongsToMany('App\Role')->as(‘role_user’);

```

源码：

```php
public function as($accessor)
{
    $this->accessor = $accessor;

    return $this;
}
```

## 关联模型的预加载

### with 函数

当作为属性访问 Eloquent 关联时，关联数据是「懒加载」的。意味着在你第一次访问该属性时，才会加载关联数据。不过，当你查询父模型时，Eloquent 还可以进行「预加载」关联数据。预加载避免了 N + 1 查询问题。

预加载可以一次操作中预加载关联模型并且自定义用于 `select` 的列，可以预加载几个不同的关联，还可以预加载嵌套关联，预加载关联数据的时候，为查询指定额外的约束条件：

```php
$books = App\Book::with(['author:id,name'])->get();

$books = App\Book::with(['author', 'publisher'])->get();

$books = App\Book::with('author.contacts')->get();

$users = App\User::with(['posts' => function ($query) {
    $query->where('title', 'like', '%first%');
}])->get();
```
我们来看看 `with` 函数：

```php
public static function with($relations)
{
    return (new static)->newQuery()->with(
        is_string($relations) ? func_get_args() : $relations
    );
}
```

预加载调用 `Eloquent/builder` 的 `with` 函数：
 
```php
public function with($relations)
{
    $eagerLoad = $this->parseWithRelations(is_string($relations) ? func_get_args() : $relations);

    $this->eagerLoad = array_merge($this->eagerLoad, $eagerLoad);

    return $this;
}

```

`eagerLoad` 成员变量用于存放预加载的关联关系，`parseWithRelations` 用于解析关联关系：

```php
protected function parseWithRelations(array $relations)
{
    $results = [];

    foreach ($relations as $name => $constraints) {
        if (is_numeric($name)) {
            $name = $constraints;

            list($name, $constraints) = Str::contains($name, ':')
                        ? $this->createSelectWithConstraint($name)
                        : [$name, function () {
                            //
                        }];
        }

        $results = $this->addNestedWiths($name, $results);

        $results[$name] = $constraints;
    }

    return $results;
}

```

当我们在模型关系中写入 `:` 符合的时候，说明我们不想 `select *`，而是想要只查询特定的字段，`createSelectWithConstraint`:

```php
protected function createSelectWithConstraint($name)
{
    return [explode(':', $name)[0], function ($query) use ($name) {
        $query->select(explode(',', explode(':', $name)[1]));
    }];
}

```

也就是为关联关系添加 `select` 条件。

当我们想要进行嵌套查询的时候，需要在关联关系中写下 `.`，`addNestedWiths` :

```php
protected function addNestedWiths($name, $results)
{
    $progress = [];

    foreach (explode('.', $name) as $segment) {
        $progress[] = $segment;

        if (! isset($results[$last = implode('.', $progress)])) {
            $results[$last] = function () {
                //
            };
        }
    }

    return $results;
}

```

可以看到，`addNestedWiths` 为嵌套的模型关系赋予默认的空闭包函数，例如 `a.b.c`，`addNestedWiths` 返回的 `results` 数组中会有三个成员: `a`、`a.b`、`a.b.c`，这三个变量的闭包函数都是空。

接下来，`parseWithRelations` 为 `a.b.c` 的闭包函数重新赋值，将用户定义的约束条件赋予给 `a.b.c`。

### get 函数预加载

`with` 函数为 `laravel` 提供了需要预加载的关联关系，`get` 函数在从数据库中获取父模型的数据后，会将需要预加载的模型也一并取出来：

```php
public function get($columns = ['*'])
{
    $builder = $this->applyScopes();

    if (count($models = $builder->getModels($columns)) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $builder->getModel()->newCollection($models);
}

```

顾名思义 `eagerLoadRelations` 函数就是获取预加载模型的的函数：

```php
public function eagerLoadRelations(array $models)
{
    foreach ($this->eagerLoad as $name => $constraints) {
        // For nested eager loads we'll skip loading them here and they will be set as an
        // eager load on the query to retrieve the relation so that they will be eager
        // loaded on that query, because that is where they get hydrated as models.
        if (strpos($name, '.') === false) {
            $models = $this->eagerLoadRelation($models, $name, $constraints);
        }
    }

    return $models;
}

```

在这里，很让人费解的是 `if` 条件，这个条件语句看起来排除了嵌套预加载关系。例如上面的 `a.b.c`，`eagerLoadRelations` 只会加载 `a` 这个关联关系。其实原因是：

> // For nested eager loads we'll skip loading them here and they will be set as an
  eager load on the query to retrieve the relation so that they will be eager
  loaded on that query, because that is where they get hydrated as models.
  
翻译过来就是说，嵌套预加载要一步一步的来，第一次只加载 `a`，获得了 `a` 的关联模型之后，第二次再加载 `b`，最后加载 `c`。这里看不懂没关系，答案在下面的代码中：

```php
protected function eagerLoadRelation(array $models, $name, Closure $constraints)
{
    $relation = $this->getRelation($name);

    $relation->addEagerConstraints($models);

    $constraints($relation);

    return $relation->match(
        $relation->initRelation($models, $name),
        $relation->getEager(), $name
    );
}

```

`eagerLoadRelation` 是预加载关联关系的核心，我们可以看到加载关联模型关系主要有四个步骤：

- 通过关系名来调用 `hasOne` 等函数来加载模型关系 `relation`
- 利用 `models` 来为模型关系添加约束条件
- 调用 `with` 函数附带的约束条件
- 从数据库获取关联模型并匹配到各个父模型中，作为父模型的属性

我们先从调用关联函数 `getRelation` 来说：

#### getRelation

```php
public function getRelation($name)
{
    $relation = Relation::noConstraints(function () use ($name) {
        try {
            return $this->getModel()->{$name}();
        } catch (BadMethodCallException $e) {
            throw RelationNotFoundException::make($this->getModel(), $name);
        }
    });

    $nested = $this->relationsNestedUnder($name);

    if (count($nested) > 0) {
        $relation->getQuery()->with($nested);
    }

    return $relation;
}
```

我们在上一个文章说过，`hasOne` 等函数会自动加约束条件例如：

```php

select phone where phone.user_id = user.id

```
但是这个约束条件并不适用于预加载，因为预加载的父模型通常不只只一个。因此需要调用函数 `noConstraints` 来避免加载约束条件:

```php
public static function noConstraints(Closure $callback)
{
    $previous = static::$constraints;

    static::$constraints = false;

    try {
        return call_user_func($callback);
    } finally {
        static::$constraints = $previous;
    }
}

```

接下来，就要调用定义关联的函数：

```php
return $this->getModel()->{$name}();
```
下面的 `relationsNestedUnder` 函数用于加载嵌套的预加载关联关系：

```php
protected function relationsNestedUnder($relation)
{
    $nested = [];

    foreach ($this->eagerLoad as $name => $constraints) {
        if ($this->isNestedUnder($relation, $name)) {
            $nested[substr($name, strlen($relation.'.'))] = $constraints;
        }
    }

    return $nested;
}

protected function isNestedUnder($relation, $name)
{
    return Str::contains($name, '.') && Str::startsWith($name, $relation.'.');
}
```
从代码上可以看出来，如果当前的模型关系是 `a`，`relationsNestedUnder` 函数会把其嵌套的关系都检测出来：`a.b`、`a.b.c`，并且放入 `nested` 数组中：`nested[b]、nested[b.c]`。

接下来：

```php
if (count($nested) > 0) {
    $relation->getQuery()->with($nested);
}
```

就会继续递归预加载关联关系。

#### 关联关系预加载约束条件

获得关联关系之后，就要加载各个关联关系自己的预加载约束条件：

```php
public function addEagerConstraints(array $models)
{
    $this->query->whereIn(
        $this->foreignKey, $this->getKeys($models, $this->localKey)
    );
}
```
也就是从父模型的外键来为关联模型添加 `where` 条件。当然各个关联关系不同，这个函数也有一定的区别。

#### with 预加载约束条件

接下来还有加载 `with` 函数的约束条件 ：

```php
$constraints($relation);
```

#### 匹配父模型

当关联关系的约束条件都设置完毕后，就要从数据库中来获取关联模型：

```php
$relation->match(
    $relation->initRelation($models, $name),
    $relation->getEager(), $name
);

public function getEager()
{
    return $this->get();
}
```
`initRelation` 会为父模型设置默认的关联模型：

```php
public function initRelation(array $models, $relation)
{
    foreach ($models as $model) {
        $model->setRelation($relation, $this->getDefaultFor($model));
    }

    return $models;
}
```
两步都做好了，接下来就要为父模型和子模型进行匹配了：

```php
public function match(array $models, Collection $results, $relation)
{
    return $this->matchOne($models, $results, $relation);
}

public function matchOne(array $models, Collection $results, $relation)
{
    return $this->matchOneOrMany($models, $results, $relation, 'one');
}

protected function matchOneOrMany(array $models, Collection $results, $relation, $type)
{
    $dictionary = $this->buildDictionary($results);

    foreach ($models as $model) {
        if (isset($dictionary[$key = $model->getAttribute($this->localKey)])) {
            $model->setRelation(
                $relation, $this->getRelationValue($dictionary, $key, $type)
            );
        }
    }

    return $models;
}
```

匹配的过程分为两步：创建目录 `buildDictionary` 和设置子模型 `setRelation`：

```php
protected function buildDictionary(Collection $results)
{
    $dictionary = [];

    $foreign = $this->getForeignKeyName();

    foreach ($results as $result) {
        $dictionary[$result->{$foreign}][] = $result;
    }

    return $dictionary;
}

```
创建目录 `buildDictionary` 函数根据子模型的外键 `foreign` 将子模型进行分类，拥有同一外键的子模型放入同一个数组中。

接下来，为父模型设置子模型：

```php
foreach ($models as $model) {
    if (isset($dictionary[$key = $model->getAttribute($this->localKey)])) {
        $model->setRelation(
            $relation, $this->getRelationValue($dictionary, $key, $type)
        );
    }
}

protected function getRelationValue(array $dictionary, $key, $type)
{
    $value = $dictionary[$key];

    return $type == 'one' ? reset($value) : $this->related->newCollection($value);
}
```
如果目录 `dictionary` 中存在父模型的主键，就会从目录中取出对应的子模型数组，并利用 `setRelation` 来为父模型设置关联模型。

## 关联模型的关联查询

### 基于存在的关联查询

官方样例：

```php
// 获得所有至少有一条评论的文章...
$posts = App\Post::has('comments')->get();

// 获得所有有三条或三条以上评论的文章...
$posts = Post::has('comments', '>=', 3)->get();

// 获得所有至少有一条获赞评论的文章...
$posts = Post::has('comments.votes')->get();

// 获得所有至少有一条评论内容满足 foo% 条件的文章
$posts = Post::whereHas('comments', function ($query) {
    $query->where('content', 'like', 'foo%');
})->get();

```

`has` 函数用于基于存在的关联查询：

```php
public function has($relation, $operator = '>=', $count = 1, $boolean = 'and', Closure $callback = null)
{
    if (strpos($relation, '.') !== false) {
        return $this->hasNested($relation, $operator, $count, $boolean, $callback);
    }

    $relation = $this->getRelationWithoutConstraints($relation);

    $method = $this->canUseExistsForExistenceCheck($operator, $count)
                    ? 'getRelationExistenceQuery'
                    : 'getRelationExistenceCountQuery';

    $hasQuery = $relation->{$method}(
        $relation->getRelated()->newQuery(), $this
    );

    if ($callback) {
        $hasQuery->callScope($callback);
    }

    return $this->addHasWhere(
        $hasQuery, $relation, $operator, $count, $boolean
    );
}

```

`has` 函数的步骤：

- 获取无约束的关联关系
- 为关联关系添加 `existence` 约束
- 为关联关系添加 `has` 外部约束
- 将关联关系添加到 `where` 条件中

#### 无约束的关联关系
```php
protected function getRelationWithoutConstraints($relation)
{
    return Relation::noConstraints(function () use ($relation) {
        return $this->getModel()->{$relation}();
    });
}
```

这个不用多说，和预加载的原理一样。

#### existence 约束

关系模型的 `existence` 约束条件很简单：

```php
select * from post where user.id = post.user_id
```

`laravel` 还考虑一种特殊情况，那就是自己关联自己，这个时候就会为模型命名一个新的 `hash`：

```php
select * from user as wedfklk where user.id = wedfklk.foreignKey
```
源代码比较简单：

```php
public function getRelationExistenceQuery(Builder $query, Builder $parentQuery, $columns = ['*'])
{
    if ($query->getQuery()->from == $parentQuery->getQuery()->from) {
        return $this->getRelationExistenceQueryForSelfRelation($query, $parentQuery, $columns);
    }

    return parent::getRelationExistenceQuery($query, $parentQuery, $columns);
}

public function getRelationExistenceQueryForSelfRelation(Builder $query, Builder $parentQuery, $columns = ['*'])
{
    $query->from($query->getModel()->getTable().' as '.$hash = $this->getRelationCountHash());

    $query->getModel()->setTable($hash);

    return $query->select($columns)->whereColumn(
        $this->getQualifiedParentKeyName(), '=', $hash.'.'.$this->getForeignKeyName()
    );
}

public function getRelationExistenceQuery(Builder $query, Builder $parentQuery, $columns = ['*'])
{
    return $query->select($columns)->whereColumn(
        $this->getQualifiedParentKeyName(), '=', $this->getExistenceCompareKey()
    );
}

public function getExistenceCompareKey()
{
    return $this->getQualifiedForeignKeyName();
}
```

#### ExistenceCount 约束

`ExistenceCount` 约束只是 `select *` 变成了 `select count(*)`:

```php
select count(*) from post where user.id = post.user_id
```
源代码：

```php
public function getRelationExistenceCountQuery(Builder $query, Builder $parentQuery)
{
    return $this->getRelationExistenceQuery(
        $query, $parentQuery, new Expression('count(*)')
    );
}
```

#### 关联关系添加到 `where` 条件

当关联关系的 `存在` 约束设置完毕后，就要加载到父模型的 `where` 条件中，一般会作为父模型的子查询：

```php
protected function addHasWhere(Builder $hasQuery, Relation $relation, $operator, $count, $boolean)
{
    $hasQuery->mergeConstraintsFrom($relation->getQuery());

    return $this->canUseExistsForExistenceCheck($operator, $count)
            ? $this->addWhereExistsQuery($hasQuery->toBase(), $boolean, $operator === '<' && $count === 1)
            : $this->addWhereCountQuery($hasQuery->toBase(), $operator, $count, $boolean);
}

public function addWhereExistsQuery(Builder $query, $boolean = 'and', $not = false)
{
    $type = $not ? 'NotExists' : 'Exists';

    $this->wheres[] = compact('type', 'operator', 'query', 'boolean');

    $this->addBinding($query->getBindings(), 'where');

    return $this;
}

protected function addWhereCountQuery(QueryBuilder $query, $operator = '>=', $count = 1, $boolean = 'and')
{
    $this->query->addBinding($query->getBindings(), 'where');

    return $this->where(
        new Expression('('.$query->toSql().')'),
        $operator,
        is_numeric($count) ? new Expression($count) : $count,
        $boolean
    );
}
```

`existence` 约束最后条件：

```php
select * from user where exists (select * from phone where phone.user_id=user.id)

```

`ExistenceCount` 约束:

```php
select * from user where (select count(*) from phone where phone.user_id=user.id) >= 3

```

#### 嵌套查询

嵌套查询需要进行递归来调用 `has` 函数：

```php
protected function hasNested($relations, $operator = '>=', $count = 1, $boolean = 'and', $callback = null)
{
    $relations = explode('.', $relations);

    $closure = function ($q) use (&$closure, &$relations, $operator, $count, $callback) {
        count($relations) > 1
            ? $q->whereHas(array_shift($relations), $closure)
            : $q->has(array_shift($relations), $operator, $count, 'and', $callback);
    };

    return $this->has(array_shift($relations), '>=', 1, $boolean, $closure);
}

public function whereHas($relation, Closure $callback = null, $operator = '>=', $count = 1)
{
    return $this->has($relation, $operator, $count, 'and', $callback);
}

```

例如 

```php
$posts = Post::has('comments.votes')->get();
```
首先 `hasNested` 会返回：

```php
$this->has('comments', '>=', 1, 'and', function ($q) use (&$closure, ‘votes’, '>=', 1, $callback) {
        $q->has(‘votes’, '>=', 1, 'and', $callback);
    }
);
```
生成的 sql:

```php
select * from post where exist (select * from comment where comment.post_id=post.id and where exist (select * from vote where vote.comment_id=comment.id))

```
### 基于不存在的关联查询

基于不存在的关联查询只是基于存在的关联查询

```php
public function doesntHave($relation, $boolean = 'and', Closure $callback = null)
{
    return $this->has($relation, '<', 1, $boolean, $callback);
}

public function whereDoesntHave($relation, Closure $callback = null)
{
    return $this->doesntHave($relation, 'and', $callback);
}
```

### 关联数据计数

如果您只想统计结果数而不需要加载实际数据，那么可以使用 withCount 方法，此方法会在您的结果集模型中添加一个 {关联名}_count 字段。例如：

```php
$posts = App\Post::withCount('comments')->get();
//select *,(select count(*) from comment where comment.post_id=post.id) as comments_count from post 

foreach ($posts as $post) {
    echo $post->comments_count;
}

//多个关联数据「计数」，并为其查询添加约束条件：
$posts = Post::withCount(['votes', 'comments' => function ($query) {
    $query->where('content', 'like', 'foo%');
}])->get();
//select *,(select count(*) from comment where comment.post_id=post.id and content like 'foo%') as comments_count,(select count(*) from votes where vote.post_id=post.id) as votes_count from post 

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;

//可以为关联数据计数结果起别名，允许在同一个关联上多次计数：
$posts = Post::withCount([
    'comments',
    'comments as pending_comments_count' => function ($query) {
        $query->where('approved', false);
    }
])->get();
//select *,(select count(*) from comment where comment.post_id=post.id) as comments_count,(select count(*) from comment where comment.post_id=post.id and approved=false) as pending_comments_count from post

echo $posts[0]->comments_count;

echo $posts[0]->pending_comments_count;
```
`withCount` 的源代码与 `has` 的代码高度相似：

```php
public function withCount($relations)
{
    if (empty($relations)) {
        return $this;
    }

    if (is_null($this->query->columns)) {
        $this->query->select([$this->query->from.'.*']);
    }

    $relations = is_array($relations) ? $relations : func_get_args();

    foreach ($this->parseWithRelations($relations) as $name => $constraints) {
        $segments = explode(' ', $name);

        unset($alias);

        if (count($segments) == 3 && Str::lower($segments[1]) == 'as') {
            list($name, $alias) = [$segments[0], $segments[2]];
        }

        $relation = $this->getRelationWithoutConstraints($name);

        $query = $relation->getRelationExistenceCountQuery(
            $relation->getRelated()->newQuery(), $this
        );

        $query->callScope($constraints);

        $query->mergeConstraintsFrom($relation->getQuery());

        $column = $alias ?? Str::snake($name.'_count');

        $this->selectSub($query->toBase(), $column);
    }

    return $this;
}

```

- 解析关联关系名称
- 获取无约束的关联关系
- 为关联关系添加 `existenceCount` 约束
- 为关联关系添加 `with` 外部约束
- 将关联关系添加到 `where` 条件中
- 设置 `alias` 别名
- 创建 `select` 子查询

### 多对多关系的中间表查询

```php
return $this->belongsToMany('App\Role')->wherePivot('approved', 1);

return $this->belongsToMany('App\Role')->wherePivotIn('priority', [1, 2]);
```

```php
public function wherePivot($column, $operator = null, $value = null, $boolean = 'and')
{
    $this->pivotWheres[] = func_get_args();

    return $this->where($this->table.'.'.$column, $operator, $value, $boolean);
}

public function wherePivotIn($column, $values, $boolean = 'and', $not = false)
{
    $this->pivotWhereIns[] = func_get_args();

    return $this->whereIn($this->table.'.'.$column, $values, $boolean, $not);
}
```
注意这里的 `pivotWheres` 与 `pivotWheres` 变量，这个变量在对中间表的加载中会被使用：

```php
protected function newPivotQuery()
{
    $query = $this->newPivotStatement();

    foreach ($this->pivotWheres as $arguments) {
        call_user_func_array([$query, 'where'], $arguments);
    }

    foreach ($this->pivotWhereIns as $arguments) {
        call_user_func_array([$query, 'whereIn'], $arguments);
    }

    return $query->where($this->foreignPivotKey, $this->parent->{$this->parentKey});
}
```