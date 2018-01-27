## 获取模型

### get 函数

```php
public function get($columns = ['*'])
{
    $builder = $this->applyScopes();

    if (count($models = $builder->getModels($columns)) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $builder->getModel()->newCollection($models);
}

public function getModels($columns = ['*'])
{
    return $this->model->hydrate(
        $this->query->get($columns)->all()
    )->all();
}
```
`get` 函数会将 `QueryBuilder` 所获取的数据进一步包装 `hydrate`。`hydrate` 函数会将数据库取回来的数据打包成数据库模型对象 `Eloquent Model`，如果可以获取到数据，还会利用函数 `eagerLoadRelations` 来预加载关系模型。

```php
public function hydrate(array $items)
{
    $instance = $this->newModelInstance();

    return $instance->newCollection(array_map(function ($item) use ($instance) {
        return $instance->newFromBuilder($item);
    }, $items));
}
```
`newModelInstance` 函数创建了一个新的数据库模型对象，重要的是这个函数为新的数据库模型对象赋予了 `connection`：

```php
public function newModelInstance($attributes = [])
{
    return $this->model->newInstance($attributes)->setConnection(
        $this->query->getConnection()->getName()
    );
}
```
`newFromBuilder` 函数会将所有数据库数据存入另一个新的 `Eloquent Model` 的 `attributes` 中：

```php
public function newFromBuilder($attributes = [], $connection = null)
{
    $model = $this->newInstance([], true);

    $model->setRawAttributes((array) $attributes, true);

    $model->setConnection($connection ?: $this->getConnectionName());

    $model->fireModelEvent('retrieved', false);

    return $model;
}
```

newInstance 函数专用于创建新的数据库对象模型：

```php
public function newInstance($attributes = [], $exists = false)
{
    $model = new static((array) $attributes);

    $model->exists = $exists;

    $model->setConnection(
        $this->getConnectionName()
    );

    return $model;
}
```
值得注意的是 `newInstance` 将 `exist` 设置为 `true`，意味着当前这个数据库模型对象是从数据库中获取而来，并非是手动新建的，这个 `exist` 为真，我们才能对这个数据库对象进行 `update`。

`setRawAttributes` 函数为新的数据库对象赋予属性值，并且进行 `sync`，标志着对象的原始状态：

```php
public function setRawAttributes(array $attributes, $sync = false)
{
    $this->attributes = $attributes;

    if ($sync) {
        $this->syncOriginal();
    }

    return $this;
}

public function syncOriginal()
{
    $this->original = $this->attributes;

    return $this;
}
```

这个原始状态的记录十分重要，原因是 `save` 函数就是利用原始值 `original` 与属性值 `attributes` 的差异来决定更新的字段。

### find 函数

`find` 函数用于利用主键 `id` 来查询数据，`find` 函数也可以传入数组，查询多个数据

```php
public function find($id, $columns = ['*'])
{
    if (is_array($id) || $id instanceof Arrayable) {
        return $this->findMany($id, $columns);
    }

    return $this->whereKey($id)->first($columns);
}

public function findMany($ids, $columns = ['*'])
{
    if (empty($ids)) {
        return $this->model->newCollection();
    }

    return $this->whereKey($ids)->get($columns);
}
```

### findOrFail 

`laravel` 还提供 `findOrFail` 函数，一般用于 `controller`，在未找到记录的时候会抛出异常。

```php
public function findOrFail($id, $columns = ['*'])
{
    $result = $this->find($id, $columns);

    if (is_array($id)) {
        if (count($result) == count(array_unique($id))) {
            return $result;
        }
    } elseif (! is_null($result)) {
        return $result;
    }

    throw (new ModelNotFoundException)->setModel(
        get_class($this->model), $id
    );
}
```

### 其他查询与数据获取方法

所用 `Query Builder` 支持的查询方法，例如 `select`、`selectSub`、`whereDate`、`whereBetween` 等等，都可以直接对 `Eloquent Model` 直接使用，程序会通过魔术方法调用 `Query Builder` 的相关方法：

```php
protected $passthru = [
    'insert', 'insertGetId', 'getBindings', 'toSql',
    'exists', 'count', 'min', 'max', 'avg', 'sum', 'getConnection',
];

public function __call($method, $parameters)
{
    ...
 
    if (in_array($method, $this->passthru)) {
        return $this->toBase()->{$method}(...$parameters);
    }

    $this->query->{$method}(...$parameters);

    return $this;
}

```

`passthru` 中的各个函数在调用前需要加载查询作用域，原因是这些操作基本上是 `aggregate` 的，需要添加搜索条件才能更加符合预期：

```php
public function toBase()
{
    return $this->applyScopes()->getQuery();
}

```

## 添加和更新模型

### save 函数

在 `Eloquent Model` 中，添加与更新模型可以统一用 `save` 函数。在添加模型的时候需要事先为 `model` 属性赋值，可以单个手动赋值，也可以批量赋值。在更新模型的时候，需要事先从数据库中取出模型，然后修改模型属性，最后执行 `save` 更新操作。官方文档：[添加和更新模型](https://d.laravel-china.org/docs/5.5/eloquent#inserting-and-updating-models) 

```php
public function save(array $options = [])
{
    $query = $this->newQueryWithoutScopes();

    if ($this->fireModelEvent('saving') === false) {
        return false;
    }

    if ($this->exists) {
        $saved = $this->isDirty() ?
                    $this->performUpdate($query) : true;
    }

    else {
        $saved = $this->performInsert($query);

        if (! $this->getConnectionName() &&
            $connection = $query->getConnection()) {
            $this->setConnection($connection->getName());
        }
    }

    if ($saved) {
        $this->finishSave($options);
    }

    return $saved;
}

```

`save` 函数不会加载全局作用域，原因是凡是利用 `save` 函数进行的插入或者更新的操作都不会存在 `where` 条件，仅仅利用自身的主键属性来进行更新。如果需要 `where` 条件可以使用 `query\builder` 的 `update` 函数，我们在下面会详细介绍：

```php
public function newQueryWithoutScopes()
{
    $builder = $this->newEloquentBuilder($this->newBaseQueryBuilder());

    return $builder->setModel($this)
                ->with($this->with)
                ->withCount($this->withCount);
}

protected function newBaseQueryBuilder()
{
    $connection = $this->getConnection();

    return new QueryBuilder(
        $connection, $connection->getQueryGrammar(), $connection->getPostProcessor()
    );
}

```
newQueryWithoutScopes 函数创建新的没有任何其他条件的 `Eloquent\builder` 类，而 `Eloquent\builder` 类需要 `Query\builder` 类作为底层查询构造器。

### performUpdate 函数

如果当前的数据库模型对象是从数据库中取出的，也就是直接或间接的调用 `get()` 函数从数据库中获取到的数据库对象，那么其 `exists` 必然是 `true`

```php
public function isDirty($attributes = null)
{
    return $this->hasChanges(
        $this->getDirty(), is_array($attributes) ? $attributes : func_get_args()
    );
}

public function getDirty()
{
    $dirty = [];

    foreach ($this->getAttributes() as $key => $value) {
        if (! $this->originalIsEquivalent($key, $value)) {
            $dirty[$key] = $value;
        }
    }

    return $dirty;
}
```
`getDirty` 函数可以获取所有与原始值不同的属性值，也就是需要更新的数据库字段。关键函数在于 `originalIsEquivalent`：

```php
protected function originalIsEquivalent($key, $current)
{
    if (! array_key_exists($key, $this->original)) {
        return false;
    }

    $original = $this->getOriginal($key);

    if ($current === $original) {
        return true;
    } elseif (is_null($current)) {
        return false;
    } elseif ($this->isDateAttribute($key)) {
        return $this->fromDateTime($current) ===
               $this->fromDateTime($original);
    } elseif ($this->hasCast($key)) {
        return $this->castAttribute($key, $current) ===
               $this->castAttribute($key, $original);
    }

    return is_numeric($current) && is_numeric($original)
            && strcmp((string) $current, (string) $original) === 0;
}
```

可以看到，对于数据库可以转化的属性都要先进行转化，然后再开始对比。比较出的结果，就是我们需要 `update` 的字段。

执行更新的时候，除了 `getDirty` 函数获得的待更新字段，还会有 `UPDATED_AT` 这个字段：

```php
protected function performUpdate(Builder $query)
{
    if ($this->fireModelEvent('updating') === false) {
        return false;
    }

    if ($this->usesTimestamps()) {
        $this->updateTimestamps();
    }

    $dirty = $this->getDirty();

    if (count($dirty) > 0) {
        $this->setKeysForSaveQuery($query)->update($dirty);

        $this->fireModelEvent('updated', false);

        $this->syncChanges();
    }

    return true;
}

protected function updateTimestamps()
{
    $time = $this->freshTimestamp();

    if (! is_null(static::UPDATED_AT) && ! $this->isDirty(static::UPDATED_AT)) {
        $this->setUpdatedAt($time);
    }

    if (! $this->exists && ! $this->isDirty(static::CREATED_AT)) {
        $this->setCreatedAt($time);
    }
}
```

执行更新的时候，`where` 条件只有一个，那就是主键 `id`：

```php
protected function setKeysForSaveQuery(Builder $query)
{
    $query->where($this->getKeyName(), '=', $this->getKeyForSaveQuery());

    return $query;
}

protected function getKeyForSaveQuery()
{
    return $this->original[$this->getKeyName()]
                    ?? $this->getKey();
}

public function getKey()
{
    return $this->getAttribute($this->getKeyName());
}
```
最后会调用 `EloquentBuilder` 的 `update` 函数：

```php
public function update(array $values)
{
    return $this->toBase()->update($this->addUpdatedAtColumn($values));
}

protected function addUpdatedAtColumn(array $values)
{
    if (! $this->model->usesTimestamps()) {
        return $values;
    }

    return Arr::add(
        $values, $this->model->getUpdatedAtColumn(),
        $this->model->freshTimestampString()
    );
}

public function freshTimestampString()
{
    return $this->fromDateTime($this->freshTimestamp());
}

public function fromDateTime($value)
{
    return is_null($value) ? $value : $this->asDateTime($value)->format(
        $this->getDateFormat()
    );
}
```

### performInsert

关于数据库对象的插入，如果数据库的主键被设置为 `increment`，也就是自增的话，程序会调用 `insertAndSetId`，这个时候不需要给数据库模型对象手动赋值主键 `id`。若果数据库的主键并不支持自增，那么就需要在插入前，为数据库对象的主键 `id` 赋值，否则数据库会报错。

```php
protected function performInsert(Builder $query)
{
    if ($this->fireModelEvent('creating') === false) {
        return false;
    }

    if ($this->usesTimestamps()) {
        $this->updateTimestamps();
    }

    $attributes = $this->attributes;

    if ($this->getIncrementing()) {
        $this->insertAndSetId($query, $attributes);
    }
    else {
        if (empty($attributes)) {
            return true;
        }

        $query->insert($attributes);
    }

    $this->exists = true;

    $this->wasRecentlyCreated = true;

    $this->fireModelEvent('created', false);

    return true;
}
```

`laravel` 默认数据库的主键支持自增属性，程序调用的也是函数 `insertAndSetId` 函数：

```php
protected function insertAndSetId(Builder $query, $attributes)
{
    $id = $query->insertGetId($attributes, $keyName = $this->getKeyName());

    $this->setAttribute($keyName, $id);
}

```
插入后，会将插入后得到的主键 `id` 返回，并赋值到模型的属性当中。

如果数据库主键不支持自增，那么我们在数据库类中要设置：

```php
public $incrementing = false;
```
每次进行插入数据的时候，需要手动给主键赋值。

### update 函数

`save` 函数仅仅支持手动的属性赋值，无法批量赋值。`laravel` 的 `Eloquent Model` 还有一个函数: `update` 支持批量属性赋值。有意思的是，`Eloquent Builder` 也有函数 `update`，那个是上一小节提到的 `performUpdate` 所调用的函数。

两个 `update` 功能一致，只是 `Model` 的 `update` 函数比较适用于更新从数据库取回的数据库对象：

```php
$flight = App\Flight::find(1);

$flight->update(['name' => 'New Flight Name','desc' => 'test']);

```
而 `Builder` 的 `update` 适用于多查询条件下的更新：

```php
App\Flight::where('active', 1)
          ->where('destination', 'San Diego')
          ->update(['delayed' => 1]);

```

无论哪一种，都会自动更新 `updated_at` 字段。

`Model` 的 `update` 函数借助 `fill` 函数与 `save` 函数：

```php
public function update(array $attributes = [], array $options = [])
{
    if (! $this->exists) {
        return false;
    }

    return $this->fill($attributes)->save($options);
}

```

### make 函数

同样的，`save` 的插入也仅仅支持手动属性赋值，如果想实现批量属性赋值的插入可以使用 `make` 函数：

```php
$model = App\Flight::make(['name' => 'New Flight Name','desc' => 'test']);

$model->save();
```

`make` 函数实际上仅仅是新建了一个 `Eloquent Model`，并批量赋予属性值：

```php
public function make(array $attributes = [])
{
    return $this->newModelInstance($attributes);
}

public function newModelInstance($attributes = [])
{
    return $this->model->newInstance($attributes)->setConnection(
        $this->query->getConnection()->getName()
    );
}

```

### create 函数

如果想要一步到位，批量赋值属性与插入一起操作，可以使用 `create` 函数：

```php
App\Flight::create(['name' => 'New Flight Name','desc' => 'test']);
```
相比较 `make` 函数，`create` 函数更进一步调用了 `save` 函数：

```php
public function create(array $attributes = [])
{
    return tap($this->newModelInstance($attributes), function ($instance) {
        $instance->save();
    });
}
```
实际上，属性值是否可以批量赋值需要受 `fillable` 或 `guarded` 来控制，如果我们想要强制批量赋值可以使用 `forceCreate`：

```php
public function forceCreate(array $attributes)
{
    return $this->model->unguarded(function () use ($attributes) {
        return $this->newModelInstance()->create($attributes);
    });
}

```

### findOrNew 函数

`laravel` 提供一种主键查询或者新建数据库对象的函数：`findOrNew`：

```php
public function findOrNew($id, $columns = ['*'])
{
    if (! is_null($model = $this->find($id, $columns))) {
        return $model;
    }

    return $this->newModelInstance();
}
```

值得注意的是，当查询失败的时候，会返回一个全新的数据库对象，不含有任何 `attributes`。

### firstOrNew 函数

`laravel` 提供一种自定义查询或者新建数据库对象的函数：`firstOrNew`：

```php
public function firstOrNew(array $attributes, array $values = [])
{
    if (! is_null($instance = $this->where($attributes)->first())) {
        return $instance;
    }

    return $this->newModelInstance($attributes + $values);
}
```
值得注意的是，如果查询失败，会返回一个含有 `attributes` 和 `values` 两者合并的属性的数据库对象。


### firstOrCreate 函数

类似于 `firstOrNew` 函数，`firstOrCreate` 函数也用于自定义查询或者新建数据库对象，不同的是，`firstOrCreate` 函数还进一步对数据进行了插入操作：

```php
public function firstOrCreate(array $attributes, array $values = [])
{
    if (! is_null($instance = $this->where($attributes)->first())) {
        return $instance;
    }

    return tap($this->newModelInstance($attributes + $values), function ($instance) {
        $instance->save();
    });
}

```

### updateOrCreate 函数

在 `firstOrCreate` 函数基础上，除了对数据进行查询，还会对查询成功的数据利用 `value` 进行更新：

```php
public function updateOrCreate(array $attributes, array $values = [])
{
    return tap($this->firstOrNew($attributes), function ($instance) use ($values) {
        $instance->fill($values)->save();
    });
}
```

### firstOr 函数

如果想要自定义查找失败后的操作，可以使用 `firstOr` 函数，该函数可以传入闭包函数，处理找不到数据的情况：

```php
public function firstOr($columns = ['*'], Closure $callback = null)
{
    if ($columns instanceof Closure) {
        $callback = $columns;

        $columns = ['*'];
    }

    if (! is_null($model = $this->first($columns))) {
        return $model;
    }

    return call_user_func($callback);
}
```

## 删除模型

删除模型也会分为两种，一种是针对 `Eloquent Model` 的删除，这种删除必须是从数据库中取出的对象。还有一种是 `Eloquent Builder` 的删除，这种删除一般会带有多个查询条件。我们这一小节主要讲 `model` 的删除：

```php
public function delete()
{
    if (is_null($this->getKeyName())) {
        throw new Exception('No primary key defined on model.');
    }

    if (! $this->exists) {
        return;
    }

    if ($this->fireModelEvent('deleting') === false) {
        return false;
    }

    $this->touchOwners();

    $this->performDeleteOnModel();

    $this->fireModelEvent('deleted', false);

    return true;
}

```

删除模型时，模型对象必然要有主键。`performDeleteOnModel` 函数执行具体的删除操作：

```php
protected function performDeleteOnModel()
{
    $this->setKeysForSaveQuery($this->newQueryWithoutScopes())->delete();

    $this->exists = false;
}

protected function setKeysForSaveQuery(Builder $query)
{
    $query->where($this->getKeyName(), '=', $this->getKeyForSaveQuery());

    return $query;
}
```
所以实际上，`Model` 调用的也是 `builder` 的 `delete` 函数。

### 软删除

如果想要使用软删除，需要使用 `Illuminate\Database\Eloquent\SoftDeletes` 这个 trait。并且需要定义软删除字段，默认为 `deleted_at`，将软删除字段放入 `dates` 中，具体用法可参考官方文档:[软删除](https://d.laravel-china.org/docs/5.5/eloquent#soft-deleting)

```php
class Flight extends Model
{
    use SoftDeletes;

    /**
     * 需要被转换成日期的属性。
     *
     * @var array
     */
    protected $dates = ['deleted_at'];
}

```
我们先看看这个 `trait`：

```php
trait SoftDeletes 
{
    public static function bootSoftDeletes()
    {
        static::addGlobalScope(new SoftDeletingScope);
    }

}
```
如果使用了软删除，在 `model` 的启动过程中，就会启动软删除的这个函数。可以看出来，软删除是需要查询作用域来合作发挥作用的。我们看看这个 `SoftDeletingScope` :

```php
class SoftDeletingScope implements Scope
{
    protected $extensions = ['Restore', 'WithTrashed', 'WithoutTrashed', 'OnlyTrashed'];
    
    public function apply(Builder $builder, Model $model)
    {
        $builder->whereNull($model->getQualifiedDeletedAtColumn());
    }

	public function extend(Builder $builder)
    {
        foreach ($this->extensions as $extension) {
            $this->{"add{$extension}"}($builder);
        }

        $builder->onDelete(function (Builder $builder) {
            $column = $this->getDeletedAtColumn($builder);

            return $builder->update([
                $column => $builder->getModel()->freshTimestampString(),
            ]);
        });
    }
}
```
`apply` 函数是加载全局域调用的函数，每次进行查询的时候，调用 `get` 函数就会自动加载这个函数，`whereNull` 这个查询条件会被加载到具体的 `where` 条件中。`deleted_at` 字段一般被设置为 `null`，在执行软删除的时候，该字段会被赋予时间格式的值，标志着被删除的时间。

在加载全局作用域的时候，还会调用 `extend` 函数，`extend` 函数为 `model` 添加了四个函数：

- WithTrashed

```php
 protected function addWithTrashed(Builder $builder)
{
    $builder->macro('withTrashed', function (Builder $builder) {
        return $builder->withoutGlobalScope($this);
    });
}
```
`withTrashed` 函数取消了软删除的全局作用域，这样我们查询数据的时候就会查询到正常数据和被软删除的数据。

- withoutTrashed

```php
protected function addWithoutTrashed(Builder $builder)
{
    $builder->macro('withoutTrashed', function (Builder $builder) {
        $model = $builder->getModel();

        $builder->withoutGlobalScope($this)->whereNull(
            $model->getQualifiedDeletedAtColumn()
        );

        return $builder;
    });
}

```
`withTrashed` 函数着重强调了不要获取软删除的数据。

- onlyTrashed

```php
protected function addOnlyTrashed(Builder $builder)
{
    $builder->macro('onlyTrashed', function (Builder $builder) {
        $model = $builder->getModel();

        $builder->withoutGlobalScope($this)->whereNotNull(
            $model->getQualifiedDeletedAtColumn()
        );

        return $builder;
    });
}
```
如果只想获取被软删除的数据，可以使用这个函数 `onlyTrashed`，可以看到，它使用了 `whereNotNull`。

- restore

```php
protected function addRestore(Builder $builder)
{
    $builder->macro('restore', function (Builder $builder) {
        $builder->withTrashed();

        return $builder->update([$builder->getModel()->getDeletedAtColumn() => null]);
    });
}

```

如果想要恢复被删除的数据，还可以使用 `restore`，重新将 `deleted_at` 数据恢复为 null。

### performDeleteOnModel

SoftDeletes 这个 trait 会重载 `performDeleteOnModel` 函数，它将不会调用 `Eloquent Builder` 的 `delete` 方法，而是采用更新操作：

```php
protected function performDeleteOnModel()
{
    if ($this->forceDeleting) {
        return $this->newQueryWithoutScopes()->where($this->getKeyName(), $this->getKey())->forceDelete();
    }

    return $this->runSoftDelete();
}

protected function runSoftDelete()
{
    $query = $this->newQueryWithoutScopes()->where($this->getKeyName(), $this->getKey());

    $time = $this->freshTimestamp();

    $columns = [$this->getDeletedAtColumn() => $this->fromDateTime($time)];

    $this->{$this->getDeletedAtColumn()} = $time;

    if ($this->timestamps) {
        $this->{$this->getUpdatedAtColumn()} = $time;

        $columns[$this->getUpdatedAtColumn()] = $this->fromDateTime($time);
    }

    $query->update($columns);
}
```
删除操作不仅更新了 `deleted_at`，还更新了 `updated_at` 字段。