# Laravel Database——Eloquent Model 更新关联模型

## 前言

在前两篇文章中，向大家介绍了定义关联关系的源码，还有基于关联关系的关联模型加载与查询的源码分析，本文开始介绍第三部分，如何利用关联关系来更新插入关联模型。

## hasOne/hasMany/MorphOne/MorphMany 更新与插入

### save 方法

正向的一对一、一对多关联保存方法用于对子模型设置外键值：

```php
public function save(Model $model)
{
    $this->setForeignAttributesForCreate($model);

    return $model->save() ? $model : false;
}

protected function setForeignAttributesForCreate(Model $model)
{
    $model->setAttribute($this->getForeignKeyName(), $this->getParentKey());
}

public function getParentKey()
{
    return $this->parent->getAttribute($this->localKey);
}
```

### saveMany 方法

```php
public function saveMany($models)
{
    foreach ($models as $model) {
        $this->save($model);
    }

    return $models;
}

```

### create 方法

`create` 方法与 `save` 方法功能一致，唯一不同的是 `create` 的参数是属性，`save` 方法的参数是 `model`。

```php
public function create(array $attributes = [])
{
    return tap($this->related->newInstance($attributes), function ($instance) {
        $this->setForeignAttributesForCreate($instance);

        $instance->save();
    });
}

protected function setForeignAttributesForCreate(Model $model)
{
    $model->setAttribute($this->getForeignKeyName(), $this->getParentKey());
}
```

### createMany 方法

```php
public function createMany(array $records)
{
    $instances = $this->related->newCollection();

    foreach ($records as $record) {
        $instances->push($this->create($record));
    }

    return $instances;
}
```

### make 方法

`make` 方法用于建立子模型对象，但是并不进行保存操作：

```php
public function make(array $attributes = [])
{
    return tap($this->related->newInstance($attributes), function ($instance) {
        $this->setForeignAttributesForCreate($instance);
    });
}
```

### update 方法

`update` 方法用于更新子模型的属性，值得注意的是时间戳的更新：

```php
public function update(array $attributes)
{
    if ($this->related->usesTimestamps()) {
        $attributes[$this->relatedUpdatedAt()] = $this->related->freshTimestampString();
    }

    return $this->query->update($attributes);
}
```

### findOrNew 方法

```php
public function findOrNew($id, $columns = ['*'])
{
    if (is_null($instance = $this->find($id, $columns))) {
        $instance = $this->related->newInstance();

        $this->setForeignAttributesForCreate($instance);
    }

    return $instance;
}
```

### firstOrCreate 方法

实际调用的是 `create` 方法：

```php
public function firstOrCreate(array $attributes, array $values = [])
{
    if (is_null($instance = $this->where($attributes)->first())) {
        $instance = $this->create($attributes + $values);
    }

    return $instance;
}
```

### updateOrCreate 方法

```php
public function updateOrCreate(array $attributes, array $values = [])
{
    return tap($this->firstOrNew($attributes), function ($instance) use ($values) {
        $instance->fill($values);

        $instance->save();
    });
}
```
## belongsTo/MorphTo 更新

### save 方法

如果我们在子模型加一个包含关联名称的 `touches` 属性后，当我们更新一个子模型时，对应父模型的 `updated_at` 字段也会被同时更新:

```php
class Comment extends Model
{
    protected $touches = ['post'];

    public function post()
    {
        return $this->belongsTo('App\Post');
    }
}


$comment = App\Comment::find(1);

$comment->text = '编辑了这条评论！';

$comment->save();

```

这是由于，对子模型调用 `save` 方法会引发 `finishSave` 函数：

```php
protected function finishSave(array $options)
{
    $this->fireModelEvent('saved', false);

    if ($this->isDirty() && ($options['touch'] ?? true)) {
        $this->touchOwners();
    }

    $this->syncOriginal();
}
```

可以看到，`touchOwners` 函数被调用：

```php
public function touchOwners()
{
    foreach ($this->touches as $relation) {
        $this->$relation()->touch();

        if ($this->$relation instanceof self) {
            $this->$relation->fireModelEvent('saved', false);

            $this->$relation->touchOwners();
        } elseif ($this->$relation instanceof Collection) {
            $this->$relation->each(function (Model $relation) {
                $relation->touchOwners();
            });
        }
    }
}
```
可以看到，`touchOwners` 函数会调用 `touch` 函数，该函数用于更新父模型的时间戳：

```php
public function touch()
{
    $column = $this->getRelated()->getUpdatedAtColumn();

    $this->rawUpdate([$column => $this->getRelated()->freshTimestampString()]);
}
```
之后，父模型还会递归调用 `touchOwners` 函数，不断更新上一级的父模型。

### update 方法

`belongsTo/MorphTo` 的更新方法用于父模型的属性更新：

```php
public function update(array $attributes)
{
    return $this->getResults()->fill($attributes)->save();
}
```

### associate 方法

如果想要更新 `belongsTo` 关联时，可以使用 `associate` 方法。此方法会在子模型中设置外键：

```php
public function associate($model)
{
    $ownerKey = $model instanceof Model ? $model->getAttribute($this->ownerKey) : $model;

    $this->child->setAttribute($this->foreignKey, $ownerKey);

    if ($model instanceof Model) {
        $this->child->setRelation($this->relation, $model);
    }

    return $this->child;
}
```

### dissociate 方法

当删除 belongsTo 关联时，可以使用 dissociate方法。此方法会设置关联外键为 null：

```php
public function dissociate()
{
    $this->child->setAttribute($this->foreignKey, null);

    return $this->child->setRelation($this->relation, null);
}
```

## belongsToMany/MorphToMany/MorphByMany 更新与插入

## attach 方法

`attach` 方法用于为多对多关系添加新的关联关系，主要进行了中间表的插入工作，用法：

```php
$user = App\User::find(1);

$user->roles()->attach($roleId);

//也可以通过传递一个数组参数向中间表写入额外数据
$user->roles()->attach($roleId, ['expires' => $expires]);

//为了方便，还允许传入 ID 数组：
$user->roles()->attach([
    1 => ['expires' => $expires],
    2 => ['expires' => $expires]
]);
```

源码：

```php
public function attach($id, array $attributes = [], $touch = true)
{
    $this->newPivotStatement()->insert($this->formatAttachRecords(
        $this->parseIds($id), $attributes
    ));

    if ($touch) {
        $this->touchIfTouching();
    }
}

protected function parseIds($value)
{
    if ($value instanceof Model) {
        return [$value->getKey()];
    }

    if ($value instanceof Collection) {
        return $value->modelKeys();
    }

    if ($value instanceof BaseCollection) {
        return $value->toArray();
    }

    return (array) $value;
}

public function newPivotStatement()
{
    return $this->query->getQuery()->newQuery()->from($this->table);
}
```

可以看到，`attach` 函数最重要的是对中间表插入新数据。

在说这段代码之前，我们要先说说多对多关联关系独有的设置：

#### 中间表 Pivot 特殊初始化设置

- 自定义中间表模型

```php
class Role extends Model
{
    /**
     * 获得此角色下的用户。
     */
    public function users()
    {
        return $this->belongsToMany('App\User')->using('App\UserRole');
    }
}
```
`using` 源码非常简单：

```php
public function using($class)
{
    $this->using = $class;

    return $this;
}
```

- 中间表时间戳字段

```php
return $this->belongsToMany('App\Role')->withTimestamps();
```
`withTimestamps` 源码：

```php
public function withTimestamps($createdAt = null, $updatedAt = null)
{
    $this->pivotCreatedAt = $createdAt;
    $this->pivotUpdatedAt = $updatedAt;

    return $this->withPivot($this->createdAt(), $this->updatedAt());
}

public function createdAt()
{
    return $this->pivotCreatedAt ?: $this->parent->getCreatedAtColumn();
}

public function updatedAt()
{
    return $this->pivotUpdatedAt ?: $this->parent->getUpdatedAtColumn();
}
```

- 中间表自定义字段

```php
return $this->belongsToMany('App\Role')->withPivot('column1', 'column2');
```
自定义字段都会存放在 `pivotColumns` 中：

```php
public function withPivot($columns)
{
    $this->pivotColumns = array_merge(
        $this->pivotColumns, is_array($columns) ? $columns : func_get_args()
    );

    return $this;
}
```

#### 中间表时间戳

我们接着说中间表的插入代码：

```php
protected function formatAttachRecords($ids, array $attributes)
{
    $records = [];

    $hasTimestamps = ($this->hasPivotColumn($this->createdAt()) ||
              $this->hasPivotColumn($this->updatedAt()));

    $attributes = $this->using
            ? $this->newPivot()->forceFill($attributes)->getAttributes()
            : $attributes;

    foreach ($ids as $key => $value) {
        $records[] = $this->formatAttachRecord(
            $key, $value, $attributes, $hasTimestamps
        );
    }

    return $records;
}

```

如果我们在设置多对多关联关系的时候，使用了时间戳，那么 `hasTimestamps` 就会为 `true`。

#### 初始化 Pivot

当我们设置了自定义的中间表模型时，就会调用 `newPivot` 函数：

```php
public function newPivot(array $attributes = [], $exists = false)
{
    $pivot = $this->related->newPivot(
        $this->parent, $attributes, $this->table, $exists, $this->using
    );

    return $pivot->setPivotKeys($this->foreignPivotKey, $this->relatedPivotKey);
}

public function newPivot(Model $parent, array $attributes, $table, $exists, $using = null)
{
    return $using ? $using::fromRawAttributes($parent, $attributes, $table, $exists)
                  : Pivot::fromAttributes($parent, $attributes, $table, $exists);
}

public function setPivotKeys($foreignKey, $relatedKey)
{
    $this->foreignKey = $foreignKey;

    $this->relatedKey = $relatedKey;

    return $this;
}
```
可以看到，`newPivot` 会返回 `Pivot` 类型的对象，另外为中间表设置了 `foreignKey` 与 `relatedKey`

#### 生成 insert 数组

```php
protected function formatAttachRecord($key, $value, $attributes, $hasTimestamps)
{
    list($id, $attributes) = $this->extractAttachIdAndAttributes($key, $value, $attributes);

    return array_merge(
        $this->baseAttachRecord($id, $hasTimestamps), $attributes
    );
}

protected function extractAttachIdAndAttributes($key, $value, array $attributes)
{
    return is_array($value)
                ? [$key, array_merge($value, $attributes)]
                : [$value, $attributes];
}
```
`extractAttachIdAndAttributes` 用于获得插入记录的主键 `id`，与其对应的属性。由于可以这样进行传入参数：

```php
$user->roles()->attach([
    1 => ['expires' => $expires],
    2 => ['expires' => $expires]
]);
```
所以要判断一下 `value` 是否是数组。`baseAttachRecord` 最终生成用于 `insert` 的属性数组：

```php
protected function baseAttachRecord($id, $timed)
{
    $record[$this->relatedPivotKey] = $id;

    $record[$this->foreignPivotKey] = $this->parent->{$this->parentKey};

    if ($timed) {
        $record = $this->addTimestampsToAttachment($record);
    }

    return $record;
}

protected function addTimestampsToAttachment(array $record, $exists = false)
{
    $fresh = $this->parent->freshTimestamp();

    if (! $exists && $this->hasPivotColumn($this->createdAt())) {
        $record[$this->createdAt()] = $fresh;
    }

    if ($this->hasPivotColumn($this->updatedAt())) {
        $record[$this->updatedAt()] = $fresh;
    }

    return $record;
}
```

#### touchIfTouching 更新多对多时间戳更新

对中间表进行插入操作后，就要对父模型与 `related` 模型进行时间戳更新操作：

```php
public function touchIfTouching()
{
    if ($this->touchingParent()) {
        $this->getParent()->touch();
    }

    if ($this->getParent()->touches($this->relationName)) {
        $this->touch();
    }
}

public function touch()
{
    if (! $this->usesTimestamps()) {
        return false;
    }

    $this->updateTimestamps();

    return $this->save();
}
```
首先，如果 `related` 模型的 `touchs` 数组中有本多对多关系，那么父模型就要进行时间戳更新操作：

```php
protected function touchingParent()
{
    return $this->getRelated()->touches($this->guessInverseRelation());
}

protected function guessInverseRelation()
{
    return Str::camel(Str::plural(class_basename($this->getParent())));
}
```

其次，如果父模型的 `touchs` 数组中存在多对多关联，那么就要进行多对多关联的 `touch` 函数，对 `related` 模型进行时间戳更新操作：

```php
public function touch()
{
    $key = $this->getRelated()->getKeyName();

    $columns = [
        $this->related->getUpdatedAtColumn() => $this->related->freshTimestampString(),
    ];

    if (count($ids = $this->allRelatedIds()) > 0) {
        $this->getRelated()->newQuery()->whereIn($key, $ids)->update($columns);
    }
}

public function allRelatedIds()
{
    return $this->newPivotQuery()->pluck($this->relatedPivotKey);
}
```

### save 方法

`belongsToMany` 的 `save` 方法用于更新多对多关系，该函数会：

- 更新 `related` 模型属性
- 在中间表中添加新的记录
- 更新父模型与 `related` 模型的时间戳

主要调用了 `attach` 函数：

```php
public function save(Model $model, array $pivotAttributes = [], $touch = true)
{
    $model->save(['touch' => false]);

    $this->attach($model->getKey(), $pivotAttributes, $touch);

    return $model;
}
```

### saveMany 方法

```php
public function saveMany($models, array $pivotAttributes = [])
{
    foreach ($models as $key => $model) {
        $this->save($model, (array) ($pivotAttributes[$key] ?? []), false);
    }

    $this->touchIfTouching();

    return $models;
}
```

### create 方法

多对多的 `create` 方法用于保存 `related` 的属性，并且可以为中间表添加 `joining` 属性信息：

```php
public function create(array $attributes = [], array $joining = [], $touch = true)
{
    $instance = $this->related->newInstance($attributes);

    $instance->save(['touch' => false]);

    $this->attach($instance->getKey(), $joining, $touch);

    return $instance;
}

```

### createMany 方法

```php
public function createMany(array $records, array $joinings = [])
{
    $instances = [];

    foreach ($records as $key => $record) {
        $instances[] = $this->create($record, (array) ($joinings[$key] ?? []), false);
    }

    $this->touchIfTouching();

    return $instances;
}
```

### detach 方法

`detach` 方法比较简单，重要的是对中间表进行删除操作：

```php
public function detach($ids = null, $touch = true)
{
    $query = $this->newPivotQuery();

    if (! is_null($ids)) {
        $ids = $this->parseIds($ids);

        if (empty($ids)) {
            return 0;
        }

        $query->whereIn($this->relatedPivotKey, (array) $ids);
    }

    $results = $query->delete();

    if ($touch) {
        $this->touchIfTouching();
    }

    return $results;
}
```
### 同步关联 sync

```php
$user->roles()->sync([1, 2, 3]);

//可以通过 ID 传递其他额外的数据到中间表：
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

源码：

```php
public function sync($ids, $detaching = true)
{
    $changes = [
        'attached' => [], 'detached' => [], 'updated' => [],
    ];

    $current = $this->newPivotQuery()->pluck(
        $this->relatedPivotKey
    )->all();

    $detach = array_diff($current, array_keys(
        $records = $this->formatRecordsList($this->parseIds($ids))
    ));

    if ($detaching && count($detach) > 0) {
        $this->detach($detach);

        $changes['detached'] = $this->castKeys($detach);
    }

    $changes = array_merge(
        $changes, $this->attachNew($records, $current, false)
    );

    if (count($changes['attached']) ||
        count($changes['updated'])) {
        $this->touchIfTouching();
    }

    return $changes;
}
```
同步关联需要删除未出现的 `id`，更新已经存在 `id`，增添新出现的 `id`。

```php
 $current = $this->newPivotQuery()->pluck(
    $this->relatedPivotKey
)->all();
```
这句用于从中间表中取出所有关联的中间表记录，并且取出 `relatedPivotKey` 值。

```php
$detach = array_diff($current, array_keys(
    $records = $this->formatRecordsList($this->parseIds($ids))
));

protected function formatRecordsList(array $records)
{
    return collect($records)->mapWithKeys(function ($attributes, $id) {
        if (! is_array($attributes)) {
            list($id, $attributes) = [$attributes, []];
        }

        return [$id => $attributes];
    })->all();
}
```

这句用于统计出待删除的中间表记录的 `relatedPivotKey` 值。

```php
if ($detaching && count($detach) > 0) {
    $this->detach($detach);

    $changes['detached'] = $this->castKeys($detach);
}
```
这句进行删除操作。

```php
$changes = array_merge(
    $changes, $this->attachNew($records, $current, false)
);

protected function attachNew(array $records, array $current, $touch = true)
{
    $changes = ['attached' => [], 'updated' => []];

    foreach ($records as $id => $attributes) {
        if (! in_array($id, $current)) {
            $this->attach($id, $attributes, $touch);

            $changes['attached'][] = $this->castKey($id);
        }

        elseif (count($attributes) > 0 &&
            $this->updateExistingPivot($id, $attributes, $touch)) {
            $changes['updated'][] = $this->castKey($id);
        }
    }

    return $changes;
}
```

对于需要新增的记录，直接调用方法 `attach` 即可。对于需要更新的记录，需要调用 `updateExistingPivot` :

```php
public function updateExistingPivot($id, array $attributes, $touch = true)
{
    if (in_array($this->updatedAt(), $this->pivotColumns)) {
        $attributes = $this->addTimestampsToAttachment($attributes, true);
    }

    $updated = $this->newPivotStatementForId($id)->update($attributes);

    if ($touch) {
        $this->touchIfTouching();
    }

    return $updated;
}

public function newPivotStatementForId($id)
{
    return $this->newPivotQuery()->where($this->relatedPivotKey, $id);
}
```

这个函数主要调用 `update` 方法。

### 切换关联 toggle

多对多关联也提供了一个 toggle 方法用于「切换」给定 IDs 的附加状态。如果给定 ID 已附加，就会被移除。同样的，如果给定 ID 已移除，就会被附加，源码：

```php
public function toggle($ids, $touch = true)
{
    $changes = [
        'attached' => [], 'detached' => [],
    ];

    $records = $this->formatRecordsList($this->parseIds($ids));

    $detach = array_values(array_intersect(
        $this->newPivotQuery()->pluck($this->relatedPivotKey)->all(),
        array_keys($records)
    ));

    if (count($detach) > 0) {
        $this->detach($detach, false);

        $changes['detached'] = $this->castKeys($detach);
    }

    $attach = array_diff_key($records, array_flip($detach));

    if (count($attach) > 0) {
        $this->attach($attach, [], false);

        $changes['attached'] = array_keys($attach);
    }

    if ($touch && (count($changes['attached']) ||
                   count($changes['detached']))) {
        $this->touchIfTouching();
    }

    return $changes;
}

```

`toggle` 函数先 `intersect` 被关联的主键，进行 `detach` 所有已经存在的记录，再 `diff` 被关联的主键，对其进行 `attach` 所有记录。

### findOrNew 方法

`findOrNew` 函数用于 `related` 模型的主键搜索与新建：

```php
public function findOrNew($id, $columns = ['*'])
{
    if (is_null($instance = $this->find($id, $columns))) {
        $instance = $this->related->newInstance();
    }

    return $instance;
}
```

### firstOrNew 方法

`firstOrNew` 函数用于 `related` 模型的属性搜索与新建：

```php
public function firstOrNew(array $attributes)
{
    if (is_null($instance = $this->where($attributes)->first())) {
        $instance = $this->related->newInstance($attributes);
    }

    return $instance;
}
```

### firstOrCreate 方法

`firstOrCreate` 函数用于 `related` 模型的属性搜索与保存，`attributes` 是 `related` 模型的搜索属性或保存属性，`joining` 是中间表属性：

```php
public function firstOrCreate(array $attributes, array $joining = [], $touch = true)
{
    if (is_null($instance = $this->where($attributes)->first())) {
        $instance = $this->create($attributes, $joining, $touch);
    }

    return $instance;
}
```

### updateOrCreate 方法

`updateOrCreate` 函数用于 `related` 模型的更新，`attributes` 是 `related` 模型的搜索属性，`values` 是 `related` 模型的更新属性，`joining` 是中间表属性：

```php
public function updateOrCreate(array $attributes, array $values = [], array $joining = [], $touch = true)
{
    if (is_null($instance = $this->where($attributes)->first())) {
        return $this->create($values, $joining, $touch);
    }

    $instance->fill($values);

    $instance->save(['touch' => false]);

    return $instance;
}
```