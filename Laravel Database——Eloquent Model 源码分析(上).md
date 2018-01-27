title: Laravel Database——Eloquent Model 源码分析(上)
tags:
  - php
  - laravel
  - database
  - eloquent
  - 源码
categories:
  - php
  - database
  - laravel
date: 2017-10-05 23:25:00
---

---
## 前言

前面几个博客向大家介绍了查询构造器的原理与源码，然而查询构造器更多是为 `Eloquent Model` 服务的，我们对数据库操作更加方便的是使用 `Eloquent Model`。 本篇文章将会大家介绍 `Model` 的一些特性原理。

## Eloquent Model 修改器

当我们在 `Eloquent` 模型实例中设置某些属性值的时候，修改器允许对 `Eloquent` 属性值进行格式化。如果对修改器不熟悉，请参考官方文档：[Eloquent： 修改器](https://d.laravel-china.org/docs/5.5/eloquent-mutators)

下面先看看修改器的原理：

```php
public function offsetSet($offset, $value)
{
    $this->setAttribute($offset, $value);
}

public function setAttribute($key, $value)
{
    if ($this->hasSetMutator($key)) {
        $method = 'set'.Str::studly($key).'Attribute';

        return $this->{$method}($value);
    }

    elseif ($value && $this->isDateAttribute($key)) {
        $value = $this->fromDateTime($value);
    }

    if ($this->isJsonCastable($key) && ! is_null($value)) {
        $value = $this->castAttributeAsJson($key, $value);
    }

    if (Str::contains($key, '->')) {
        return $this->fillJsonAttribute($key, $value);
    }

    $this->attributes[$key] = $value;

    return $this;
}

```
### 自定义修改器

当我们为 `model` 的成员变量赋值的时候，就会调用 `offsetSet` 函数，进而运行 `setAttribute` 函数，在这个函数中第一个检查的就是是否存在预处理函数：

```php
public function hasSetMutator($key)
{
    return method_exists($this, 'set'.Str::studly($key).'Attribute');
}
```

如果存在该函数，就会直接调用自定义修改器。

### 日期转换器

接着如果没有自定义修改器的话，还会检查当前更新的成员变量是否是日期属性：

```php
protected function isDateAttribute($key)
{
    return in_array($key, $this->getDates()) ||
                                $this->isDateCastable($key);
}

public function getDates()
{
    $defaults = [static::CREATED_AT, static::UPDATED_AT];

    return $this->usesTimestamps()
                ? array_unique(array_merge($this->dates, $defaults))
                : $this->dates;
}

protected function isDateCastable($key)
{
    return $this->hasCast($key, ['date', 'datetime']);
}
```
字段的时间属性有两种设置方法，一种是设置 `$dates` 属性：

```php
protected $dates = ['date_attr'];
```
还有一种方法是设置 `cast` 数组：

```php
protected $casts = ['date_attr' => 'date'];
```

只要是时间属性的字段，无论是什么类型的值，`laravel` 都会自动将其转化为数据库的时间格式。数据库的时间格式设置是 `dateFormat` 成员变量，不设置的时候，默认的时间格式为 `Y-m-d H:i:s':

```php
protected $dateFormat = ['U'];

protected $dateFormat = ['Y-m-d H:i:s'];
```
当数据库对应的字段是时间类型时，为其赋值就可以非常灵活。我们可以赋值 `Carbon` 类型、`DateTime` 类型、数字类型、字符串等等：

```php
public function fromDateTime($value)
{
    return is_null($value) ? $value : $this->asDateTime($value)->format(
        $this->getDateFormat()
    );
}

protected function asDateTime($value)
{
    if ($value instanceof Carbon) {
        return $value;
    }

    if ($value instanceof DateTimeInterface) {
        return new Carbon(
            $value->format('Y-m-d H:i:s.u'), $value->getTimezone()
        );
    }

    if (is_numeric($value)) {
        return Carbon::createFromTimestamp($value);
    }

    if ($this->isStandardDateFormat($value)) {
        return Carbon::createFromFormat('Y-m-d', $value)->startOfDay();
    }

    return Carbon::createFromFormat(
        $this->getDateFormat(), $value
    );
}
```
### json 转换器

接下来，如果该变量被设置为 `array`、`json` 等属性，那么其将会转化为 `json` 类型。

```php
protected function isJsonCastable($key)
{
    return $this->hasCast($key, ['array', 'json', 'object', 'collection']);
}

protected function asJson($value)
{
    return json_encode($value);
}
```

## Eloquent Model 访问器

相比较修改器来说，访问器的适用情景会更加多。例如，我们经常把一些关于类型的字段设置为 `1`、`2`、`3` 等等，例如用户数据表中用户性别字段，`1` 代表男，`2` 代表女，很多时候我们取出这些值之后必然要经过转换，然后再显示出来。这时候就需要定义访问器。

访问器的源码：

```php
public function getAttribute($key)
{
    if (! $key) {
        return;
    }

    if (array_key_exists($key, $this->attributes) ||
        $this->hasGetMutator($key)) {
        return $this->getAttributeValue($key);
    }

    if (method_exists(self::class, $key)) {
        return;
    }

    return $this->getRelationValue($key);
}

```

可以看到，当我们访问数据库对象的成员变量的时候，大致可以分为两类：属性值与关系对象。关系对象我们以后再详细来说，本文中先说关于属性的访问。

```php
public function getAttributeValue($key)
{
    $value = $this->getAttributeFromArray($key);

    if ($this->hasGetMutator($key)) {
        return $this->mutateAttribute($key, $value);
    }

    if ($this->hasCast($key)) {
        return $this->castAttribute($key, $value);
    }

    if (in_array($key, $this->getDates()) &&
        ! is_null($value)) {
        return $this->asDateTime($value);
    }

    return $value;
}

```

与修改器类似，访问器也由三部分构成：自定义访问器、日期访问器、类型访问器。

### 获取原始值

访问器的第一步就是从成员变量 `attributes` 中获取原始的字段值，一般指的是存在数据库的值。有的时候，我们要取的属性并不在 `attributes` 中，这时候就会返回 `null`。

```php
protected function getAttributeFromArray($key)
{
    if (isset($this->attributes[$key])) {
        return $this->attributes[$key];
    }
}
```

### 自定义访问器

如果定义了访问器，那么就会调用访问器，获取返回值：

```php
public function hasGetMutator($key)
{
    return method_exists($this, 'get'.Str::studly($key).'Attribute');
}

protected function mutateAttribute($key, $value)
{
    return $this->{'get'.Str::studly($key).'Attribute'}($value);
}

```

### 类型转换

若我们在成员变量 `$casts` 数组中为属性定义了类型转换，那么就要进行类型转换：

```php
public function hasCast($key, $types = null)
{
    if (array_key_exists($key, $this->getCasts())) {
        return $types ? in_array($this->getCastType($key), (array) $types, true) : true;
    }

    return false;
}

protected function castAttribute($key, $value)
{
    if (is_null($value)) {
        return $value;
    }

    switch ($this->getCastType($key)) {
        case 'int':
        case 'integer':
            return (int) $value;
        case 'real':
        case 'float':
        case 'double':
            return (float) $value;
        case 'string':
            return (string) $value;
        case 'bool':
        case 'boolean':
            return (bool) $value;
        case 'object':
            return $this->fromJson($value, true);
        case 'array':
        case 'json':
            return $this->fromJson($value);
        case 'collection':
            return new BaseCollection($this->fromJson($value));
        case 'date':
            return $this->asDate($value);
        case 'datetime':
            return $this->asDateTime($value);
        case 'timestamp':
            return $this->asTimestamp($value);
        default:
            return $value;
    }
}

```

### 日期转换

若当前属性是 `CREATED_AT`、`UPDATED_AT` 或者被存入成员变量 `dates` 中，那么就要进行日期转换。日期转换函数 `asDateTime` 可以查看上一节中的内容。

## Eloquent Model 数组转化

在使用数据库对象中，我们经常使用 `toArray` 函数，它可以将从数据库中取出的所有属性和关系模型转化为数组：

```php
public function toArray()
{
    return array_merge($this->attributesToArray(), $this->relationsToArray());
}
``` 

本文中只介绍属性转化为数组的部分：

```php
public function attributesToArray()
{
    $attributes = $this->addDateAttributesToArray(
        $attributes = $this->getArrayableAttributes()
    );

    $attributes = $this->addMutatedAttributesToArray(
        $attributes, $mutatedAttributes = $this->getMutatedAttributes()
    );
    
    $attributes = $this->addCastAttributesToArray(
        $attributes, $mutatedAttributes
    );

    foreach ($this->getArrayableAppends() as $key) {
        $attributes[$key] = $this->mutateAttributeForArray($key, null);
    }

    return $attributes;
}
```
与访问器与修改器类似，需要转为数组的元素有日期类型、自定义访问器、类型转换，我们接下来一个个看：

### getArrayableAttributes 原始值获取

首先我们要从成员变量 `attributes` 数组中获取原始值：

```php
protected function getArrayableAttributes()
{
    return $this->getArrayableItems($this->attributes);
}

protected function getArrayableItems(array $values)
{
    if (count($this->getVisible()) > 0) {
        $values = array_intersect_key($values, array_flip($this->getVisible()));
    }

    if (count($this->getHidden()) > 0) {
        $values = array_diff_key($values, array_flip($this->getHidden()));
    }

    return $values;
}

```
我们还可以为数据库对象设置可见元素 `$visible` 与隐藏元素 `$hidden`，这两个变量会控制 `toArray` 可转化的元素属性。

### 日期转换

```php
protected function addDateAttributesToArray(array $attributes)
{
    foreach ($this->getDates() as $key) {
        if (! isset($attributes[$key])) {
            continue;
        }

        $attributes[$key] = $this->serializeDate(
            $this->asDateTime($attributes[$key])
        );
    }

    return $attributes;
}

protected function serializeDate(DateTimeInterface $date)
{
    return $date->format($this->getDateFormat());
}

```

### 自定义访问器转换

定义了自定义访问器的属性，会调用访问器函数来覆盖原有的属性值，首先我们需要获取所有的自定义访问器变量：

```php
public function getMutatedAttributes()
{
    $class = static::class;

    if (! isset(static::$mutatorCache[$class])) {
        static::cacheMutatedAttributes($class);
    }

    return static::$mutatorCache[$class];
}

public static function cacheMutatedAttributes($class)
{
    static::$mutatorCache[$class] = collect(static::getMutatorMethods($class))->map(function ($match) {
        return lcfirst(static::$snakeAttributes ? Str::snake($match) : $match);
    })->all();
}

protected static function getMutatorMethods($class)
{
    preg_match_all('/(?<=^|;)get([^;]+?)Attribute(;|$)/', implode(';', get_class_methods($class)), $matches);

    return $matches[1];
}
```
可以看到，函数用 `get_class_methods` 获取类内所有的函数，并筛选出符合 `get...Attribute` 的函数，获得自定义的访问器变量，并缓存到 `mutatorCache` 中。

接着将会利用自定义访问器变量替换原始值：

```php
protected function addMutatedAttributesToArray(array $attributes, array $mutatedAttributes)
{
    foreach ($mutatedAttributes as $key) {
        if (! array_key_exists($key, $attributes)) {
            continue;
        }

        $attributes[$key] = $this->mutateAttributeForArray(
            $key, $attributes[$key]
        );
    }

    return $attributes;
}

protected function mutateAttributeForArray($key, $value)
{
    $value = $this->mutateAttribute($key, $value);

    return $value instanceof Arrayable ? $value->toArray() : $value;
}

```
### cast 类型转换

被定义在 `cast` 数组中的变量也要进行数组转换，调用的方法和访问器相同，也是 `castAttribute`，如果是时间类型，还要按照时间格式来转换：

```php
protected function addCastAttributesToArray(array $attributes, array $mutatedAttributes)
{
    foreach ($this->getCasts() as $key => $value) {
        if (! array_key_exists($key, $attributes) || in_array($key, $mutatedAttributes)) {
            continue;
        }

        $attributes[$key] = $this->castAttribute(
            $key, $attributes[$key]
        );

        if ($attributes[$key] &&
            ($value === 'date' || $value === 'datetime')) {
            $attributes[$key] = $this->serializeDate($attributes[$key]);
        }
    }

    return $attributes;
}

```

### appends 额外属性添加

`toArray()` 还会将我们定义在 `appends` 变量中的属性一起进行数组转换，但是注意被放入 `appends` 成员变量数组中的属性需要有自定义访问器函数：

```php
protected function getArrayableAppends()
{
    if (! count($this->appends)) {
        return [];
    }

    return $this->getArrayableItems(
        array_combine($this->appends, $this->appends)
    );
}
```

## 查询作用域

查询作用域分为全局作用域与本地作用域。全局作用域不需要手动调用，由程序在每次的查询中自动加载，本地作用域需要在查询的时候进行手动调用。官方文档：[查询作用域](https://d.laravel-china.org/docs/5.5/eloquent#query-scopes)

### 全局作用域

一般全局作用域需要定义一个实现 `Illuminate\Database\Eloquent\Scope` 接口的类，该接口要求你实现一个方法：`apply`。需要的话可以在 `apply` 方法中添加 `where` 条件到查询。

要将全局作用域分配给模型，需要重写给定模型的 `boot` 方法并使用 `addGlobalScope` 方法。

另外，我们还可以向 `addGlobalScope` 中添加匿名函数实现匿名全局作用域。

我们先看看源码：

```php
public static function addGlobalScope($scope, Closure $implementation = null)
{
    if (is_string($scope) && ! is_null($implementation)) {
        return static::$globalScopes[static::class][$scope] = $implementation;
    } elseif ($scope instanceof Closure) {
        return static::$globalScopes[static::class][spl_object_hash($scope)] = $scope;
    } elseif ($scope instanceof Scope) {
        return static::$globalScopes[static::class][get_class($scope)] = $scope;
    }

    throw new InvalidArgumentException('Global scope must be an instance of Closure or Scope.');
}

```

可以看到，全局作用域使用的是全局的静态变量 `globalScopes`，该变量保存着所有数据库对象的全局作用域。

`Eloquent\Model` 类并不负责查询功能，相关功能由 `Eloquent\Builder` 负责，因此每次查询都会间接调用 `Eloquent\Builder` 类。

```php
public function __call($method, $parameters)
{
    if (in_array($method, ['increment', 'decrement'])) {
        return $this->$method(...$parameters);
    }

    try {
        return $this->newQuery()->$method(...$parameters);
    } catch (BadMethodCallException $e) {
        throw new BadMethodCallException(
            sprintf('Call to undefined method %s::%s()', get_class($this), $method)
        );
    }
}
```
创建新的 `Eloquent\Builder` 类需要 `newQuery` 函数：

```php
public function newQuery()
{
    $builder = $this->newQueryWithoutScopes();

    foreach ($this->getGlobalScopes() as $identifier => $scope) {
        $builder->withGlobalScope($identifier, $scope);
    }

    return $builder;
}

public function getGlobalScopes()
{
    return Arr::get(static::$globalScopes, static::class, []);
}

public function withGlobalScope($identifier, $scope)
{
    $this->scopes[$identifier] = $scope;

    if (method_exists($scope, 'extend')) {
        $scope->extend($this);
    }

    return $this;
}
```
`newQuery` 函数为 `Eloquent\builder` 加载全局作用域，这样静态变量 `globalScopes` 的值就会被赋到 `Eloquent\builder` 的 `scopes` 成员变量中。

当我们使用 `get()` 函数获取数据库数据的时候，也需要借助魔术方法调用 `Illuminate\Database\Eloquent\Builder` 类的 `get` 函数：

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

调用 `applyScopes` 函数加载所有的全局作用域：

```php
public function applyScopes()
{
    if (! $this->scopes) {
        return $this;
    }

    $builder = clone $this;

    foreach ($this->scopes as $identifier => $scope) {
        if (! isset($builder->scopes[$identifier])) {
            continue;
        }

        $builder->callScope(function (Builder $builder) use ($scope) {
            if ($scope instanceof Closure) {
                $scope($builder);
            }

            if ($scope instanceof Scope) {
                $scope->apply($builder, $this->getModel());
            }
        });
    }

    return $builder;
}
```
可以看到，`builder` 查询类会通过 `callScope` 加载全局作用域的查询条件。

```php
protected function callScope(callable $scope, $parameters = [])
{
    array_unshift($parameters, $this);

    $query = $this->getQuery();

    $originalWhereCount = is_null($query->wheres)
                ? 0 : count($query->wheres);

    $result = $scope(...array_values($parameters)) ?? $this;

    if (count((array) $query->wheres) > $originalWhereCount) {
        $this->addNewWheresWithinGroup($query, $originalWhereCount);
    }

    return $result;
}
```

`callScope` 函数首先会获取更加底层的 `Query\builder`，更新 `query\bulid` 的 `where` 条件。

`addNewWheresWithinGroup` 这个函数很重要，它为 `Query\builder` 提供 `nest` 类型的 `where` 条件：

```php
protected function addNewWheresWithinGroup(QueryBuilder $query, $originalWhereCount)
{
    $allWheres = $query->wheres;

    $query->wheres = [];

    $this->groupWhereSliceForScope(
        $query, array_slice($allWheres, 0, $originalWhereCount)
    );

    $this->groupWhereSliceForScope(
        $query, array_slice($allWheres, $originalWhereCount)
    );
}

protected function groupWhereSliceForScope(QueryBuilder $query, $whereSlice)
{
    $whereBooleans = collect($whereSlice)->pluck('boolean');

    if ($whereBooleans->contains('or')) {
        $query->wheres[] = $this->createNestedWhere(
            $whereSlice, $whereBooleans->first()
        );
    } else {
        $query->wheres = array_merge($query->wheres, $whereSlice);
    }
}

protected function createNestedWhere($whereSlice, $boolean = 'and')
{
    $whereGroup = $this->getQuery()->forNestedWhere();

    $whereGroup->wheres = $whereSlice;

    return ['type' => 'Nested', 'query' => $whereGroup, 'boolean' => $boolean];
}
```
当我们在查询作用域中，所有的查询条件连接符都是 `and` 的时候，可以直接合并到 `where` 中。

如果我们在查询作用域中或者原查询条件写下了 `orWhere`、`orWhereColumn` 等等连接符为 `or` 的查询条件，那么就会利用 `createNestedWhere` 函数创建 `nest` 类型的 `where` 条件。这个 `where` 条件会包含查询作用域的所有查询条件，或者原查询的所有查询条件。

### 本地作用域

全局作用域会自定加载到所有的查询条件当中，`laravel` 中还有本地作用域，只有在查询时调用才会生效。

本地作用域是由魔术方法 `__call` 实现的：

```php
public function __call($method, $parameters)
{
    ...
    
    if (method_exists($this->model, $scope = 'scope'.ucfirst($method))) {
        return $this->callScope([$this->model, $scope], $parameters);
    }

    if (in_array($method, $this->passthru)) {
        return $this->toBase()->{$method}(...$parameters);
    }

    $this->query->{$method}(...$parameters);

    return $this;
}

```
### 批量调用本地作用域

`laravel` 还提供一个方法可以一次性调用多个本地作用域：

```php
$scopes = [
    'published',
    'category' => 'Laravel',
    'framework' => ['Laravel', '5.3'],
];

(new EloquentModelStub)->scopes($scopes);
```

上面的写法会调用三个本地作用域，它们的参数是 `$scopes` 的值。

```php
public function scopes(array $scopes)
{
    $builder = $this;

    foreach ($scopes as $scope => $parameters) {
        if (is_int($scope)) {
            list($scope, $parameters) = [$parameters, []];
        }

        $builder = $builder->callScope(
            [$this->model, 'scope'.ucfirst($scope)],
            (array) $parameters
        );
    }

    return $builder;
}

```

## fill 批量赋值

`Eloquent Model` 默认只能一个一个的设置数据库对象的属性，这是为了保护数据库。但是有的时候，字段过多会造成代码很繁琐。因此，`laravel` 提供属性批量赋值的功能，`fill` 函数，相关的官方文档：[批量赋值](https://d.laravel-china.org/docs/5.5/eloquent#mass-assignment)

### fill 函数

```php
public function fill(array $attributes)
{
    $totallyGuarded = $this->totallyGuarded();

    foreach ($this->fillableFromArray($attributes) as $key => $value) {
        $key = $this->removeTableFromKey($key);

        if ($this->isFillable($key)) {
            $this->setAttribute($key, $value);
        } elseif ($totallyGuarded) {
            throw new MassAssignmentException($key);
        }
    }

    return $this;
}

```
`fill` 函数会从参数 `attributes` 中选取可以批量赋值的属性。所谓的可以批量赋值的属性，是指被 `fillable` 或 `guarded` 成员变量设置的参数。被放入 `fillable` 的属性允许批量赋值的属性，被放入 `guarded` 的属性禁止批量赋值。

获取可批量赋值的属性：

```php
protected function fillableFromArray(array $attributes)
{
    if (count($this->getFillable()) > 0 && ! static::$unguarded) {
        return array_intersect_key($attributes, array_flip($this->getFillable()));
    }

    return $attributes;
}

public function getFillable()
{
    return $this->fillable;
}
``` 
可以看到，若想要实现批量赋值，需要将属性设置在 `fillable` 成员数组中。

在 `laravel` 中，有一种数据库对象关系是 `morph`，也就是 `多态` 关系，这种关系也会调用 `fill` 函数，这个时候传入的参数 `attributes` 会带有数据库前缀。接下来，就要调用 `removeTableFromKey` 函数来去除数据库前缀：

```php
protected function removeTableFromKey($key)
{
    return Str::contains($key, '.') ? last(explode('.', $key)) : $key;
}
```
下一步，还要进一步验证属性的 `fillable`：

```php
public function isFillable($key)
{
    if (static::$unguarded) {
        return true;
    }

    if (in_array($key, $this->getFillable())) {
        return true;
    }

    if ($this->isGuarded($key)) {
        return false;
    }

    return empty($this->getFillable()) &&
        ! Str::startsWith($key, '_');
}

```
如果当前 `unguarded` 开启，也就是不会保护任何属性，那么直接返回 `true`。如果当前属性在 `fillable` 中，也会返回 `true`。如果当前属性在 `guarded` 中，返回 `false`。最后，如果 `fillable` 是空数组，也会返回 `true`。

### forceFill

如果不想受 `fillable` 或者 `guarded` 等的影响，还可以使用 `forceFill` 强制来批量赋值。

```php
public function forceFill(array $attributes)
{
    return static::unguarded(function () use ($attributes) {
        return $this->fill($attributes);
    });
}

public static function unguarded(callable $callback)
{
    if (static::$unguarded) {
        return $callback();
    }

    static::unguard();

    try {
        return $callback();
    } finally {
        static::reguard();
    }
}
```




