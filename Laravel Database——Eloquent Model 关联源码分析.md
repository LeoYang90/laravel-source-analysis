# Laravel Database——Eloquent Model 关联源码分析

## 前言

数据库表通常相互关联。`laravel` 中的模型关联功能使得关于数据库的关联代码变得更加简单，更加优雅。本文会详细说说关于模型关联的源码，以便更好的理解和使用关联模型。官方文档：[Eloquent：关联](https://d.laravel-china.org/docs/5.5/eloquent-relationships#defining-relationships)

## 定义关联
所谓的定义关联，就是在一个 `Model` 中定义一个关联函数，我们利用这个关联函数去操作另外一个 `Model`，例如，`user` 表是用户表，`posts` 是用户发的文章，一个用户可以发表多篇文章，我们就可以这样写：

```php
$user->posts()->where('active', 1)->get();
```
这表明了我们想通过 `$user` 这个用户查询到状态 `active` 为 1 的所有文章，`posts` 就是关联函数，我们可以通过这个关联函数去操作另一个与 `user` 关联的表。

在说模型关联的定义之前，我们要先说说父模型与子模型的概念。所谓的父模型是指在模型关系中主动的一方，例如用户模型和文章模型中的用户，相应的子模型就是模型关系中的被动一方，例如文章模型。在正向定义中，被关联的是子模型，而在反向关联中，被关联的是父模型。

我们知道，关联有多种形式，各种关系如下：

![](http://owql68l6p.bkt.clouddn.com/scheme.png)

## hasOne 一对一

我们以官方文档的例子来说明，一个 `User` 模型可能关联一个 `Phone` 模型：

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
我们来看看 `hasOne` 的源码：

```php
public function hasOne($related, $foreignKey = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    $foreignKey = $foreignKey ?: $this->getForeignKey();

    $localKey = $localKey ?: $this->getKeyName();

    return new HasOne($instance->newQuery(), $this, $instance->getTable().'.'.$foreignKey, $localKey);
}
```
`newRelatedInstance` 函数负责建立一个新的被关联的模型实例，主要目的是设置数据库连接：

```php
protected function newRelatedInstance($class)
{
    return tap(new $class, function ($instance) {
        if (! $instance->getConnectionName()) {
            $instance->setConnection($this->connection);
        }
    });
}

```

在一对一的关系中，`foreignKey` 外键名默认是父模型的类名和主键名的蛇形变量，`localKey` 是父模型的主键名：

```php
public function getForeignKey()
{
    return Str::snake(class_basename($this)).'_'.$this->primaryKey;
}
```
 
`hasOne` 函数的构造函数继承 `HasOneOrMany` 类，也就是说，一对一与一对多构造函数相同，这部分主要设置外键名：

```php
public function __construct(Builder $query, Model $parent, $foreignKey, $localKey)
{
    $this->localKey = $localKey;
    $this->foreignKey = $foreignKey;

    parent::__construct($query, $parent);
}
```
`HasOneOrMany` 类继承 `Relation` 类，这部分主要设置 `parent` (父模型)、被关联模型（子模型）与被关联模型（子模型）的查询构造器：

```php
public function __construct(Builder $query, Model $parent)
{
    $this->query = $query;
    $this->parent = $parent;
    $this->related = $query->getModel();

    $this->addConstraints();
}
```
`hasOne` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/hasOne%27.png)

除了保存被关联模型的查询构造器、被关联模型与 `parent` 模型之外，还会提供额外的限制条件：

```php
public function addConstraints()
{
    if (static::$constraints) {
        $this->query->where($this->foreignKey, '=', $this->getParentKey());

        $this->query->whereNotNull($this->foreignKey);
    }
}

public function getParentKey()
{
    return $this->parent->getAttribute($this->localKey);
}
```

限制条件为被关联模型和关联模型建立外键约束关系：

```php
select phone where phone.user_id = 1 (user.id)
```

## hasMany 一对多

在模型关联的定义中，一对一与一对多源码是一样的：

```php
public function hasMany($related, $foreignKey = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    $foreignKey = $foreignKey ?: $this->getForeignKey();

    $localKey = $localKey ?: $this->getKeyName();

    return new HasMany(
        $instance->newQuery(), $this, $instance->getTable().'.'.$foreignKey, $localKey
    );
}
```
`hasMany ` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/hasMany.png)

限制条件与一对一相同，为被关联模型和关联模型建立外键约束关系：

```php
select phone where phone.user_id = 1 (user.id)
```

## belongsTo 一对一、一对多 反向关联
如果想要从文章反向查找作者用户，那么可以定义反向关联：

```php
public function user()
{
    return $this->belongsTo('App\User', 'foreign_key', 'other_key');
}
```

`belongsTo` 源码：

```php
public function belongsTo($related, $foreignKey = null, $ownerKey = null, $relation = null)
{
    if (is_null($relation)) {
        $relation = $this->guessBelongsToRelation();
    }

    $instance = $this->newRelatedInstance($related);

    if (is_null($foreignKey)) {
        $foreignKey = Str::snake($relation).'_'.$instance->getKeyName();
    }

    $ownerKey = $ownerKey ?: $instance->getKeyName();

    return new BelongsTo(
        $instance->newQuery(), $this, $foreignKey, $ownerKey, $relation
    );
}

```
正向定义与反向定义不同的是多了一个参数 `relation`，这个参数默认值是从 `debug_backtrace` 函数获取的：

```php
protected function guessBelongsToRelation()
{
    list($one, $two, $caller) = debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 3);

    return $caller['function'];
}
```
也就是我们的关联函数名 `user`，`belongsTo` 函数会将关联函数名作为关联名保存起来。

另一个不同是外键的默认名称，不再是类名 + 主键名，而是关联名 + 主键名：

```php
if (is_null($foreignKey)) {
    $foreignKey = Str::snake($relation).'_'.$instance->getKeyName();
}
```

我们接着看 `belongsTo` 函数：

```php
public function __construct(Builder $query, Model $child, $foreignKey, $ownerKey, $relation)
{
    $this->ownerKey = $ownerKey;
    $this->relation = $relation;
    $this->foreignKey = $foreignKey;

    $this->child = $child;

    parent::__construct($query, $child);
}

```

我们可以看出来，相对于正向关联，反向关联除了保存外键名与主键名之外，还保存了关系名、子模型。值得注意的是，反向关联中 `related` 代表父模型，`parent` 代表子模型，与正向关联相反。

`hasMany ` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/belongsTo1.png)

约束条件也相应地进行反转改变：

```php
public function addConstraints()
{
    if (static::$constraints) {
        $table = $this->related->getTable();

        $this->query->where($table.'.'.$this->ownerKey, '=', $this->child->{$this->foreignKey});
    }
}

```

限制条件：

```php
select user where user.id = 1 (post.user_id)
```

## belongsMany 多对多

多对多关系由于中间表的原因相对来说比较复杂，涉及的参数也非常多。我们以官网例子：

```php
class User extends Model
{
    /**
     * 获得此用户的角色。
     */
    public function roles()
    {
        return $this->belongsToMany('App\Role', 'role_user', 'user_id', 'role_id');
    }
}
```

`User` 表与 `role` 表是多对多关系，另外有一中间表 `user_role` 表，我们在定义关系的时候，`related` 是被关联模型，`table` 是中间表，`foreignPivotKey` 是中间表中父模型外键名，`relatedPivotKey` 是中间表中子模型外键名，`parentKey` 是父模型主键名，`relatedKey` 是子模型主键名，`relation` 是关系名。

```php
public function belongsToMany($related, $table = null, $foreignPivotKey = null, $relatedPivotKey = null, $parentKey = null, $relatedKey = null, $relation = null)
{
    if (is_null($relation)) {
        $relation = $this->guessBelongsToManyRelation();
    }

    $instance = $this->newRelatedInstance($related);

    $foreignPivotKey = $foreignPivotKey ?: $this->getForeignKey();

    $relatedPivotKey = $relatedPivotKey ?: $instance->getForeignKey();

    if (is_null($table)) {
        $table = $this->joiningTable($related);
    }

    return new BelongsToMany(
        $instance->newQuery(), $this, $table, $foreignPivotKey,
        $relatedPivotKey, $parentKey ?: $this->getKeyName(),
        $relatedKey ?: $instance->getKeyName(), $relation
    );
}

```

获取关联名称仍然使用的是 `debug_backtrace` 函数，不同于`guessBelongsToRelation ` 函数只有 `belongsTo` 调用, `guessBelongsToManyRelation` 函数还可以被 `morphedByMany` 函数调用，所以不能单纯的限制返回堆栈帧：

```php
public static $manyMethods = [
    'belongsToMany', 'morphToMany', 'morphedByMany',
    'guessBelongsToManyRelation', 'findFirstMethodThatIsntRelation',
];
    
protected function guessBelongsToManyRelation()
{
    $caller = Arr::first(debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS), function ($trace) {
        return ! in_array($trace['function'], Model::$manyMethods);
    });

    return ! is_null($caller) ? $caller['function'] : null;
}

```
默认的中间表是两个表名的蛇形变量：

```php
public function joiningTable($related)
{
    $models = [
        Str::snake(class_basename($related)),
        Str::snake(class_basename($this)),
    ];

    sort($models);

    return strtolower(implode('_', $models));
}

```
`BelongsToMany` 的初始化也需要保存这些变量：

```php
public function __construct(Builder $query, Model $parent, $table, $foreignPivotKey,
                                $relatedPivotKey, $parentKey, $relatedKey, $relationName = null)
{
    $this->table = $table;
    $this->parentKey = $parentKey;
    $this->relatedKey = $relatedKey;
    $this->relationName = $relationName;
    $this->relatedPivotKey = $relatedPivotKey;
    $this->foreignPivotKey = $foreignPivotKey;

    parent::__construct($query, $parent);
}

```

`belongsToMany` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/belongsMany1.png)

反向的多对多模型关系：

![](http://owql68l6p.bkt.clouddn.com/belongsMany2.png)

限制条件：

```php
public function addConstraints()
{
    $this->performJoin();

    if (static::$constraints) {
        $this->addWhereConstraints();
    }
}

protected function performJoin($query = null)
{
    $query = $query ?: $this->query;

    $baseTable = $this->related->getTable();

    $key = $baseTable.'.'.$this->relatedKey;

    $query->join($this->table, $key, '=', $this->getQualifiedRelatedPivotKeyName());

    return $this;
}

protected function addWhereConstraints()
{
    $this->query->where(
        $this->getQualifiedForeignPivotKeyName(), '=', $this->parent->{$this->parentKey}
    );

    return $this;
}
```

本例中的 where 条件：

```php
select role join role_user on role_user.role_id = 1 (role.id)

select role where role_user.user_id = 1 (user.id)

```

## hasManyThrough 远程一对多

远层一对多 关联提供了方便、简短的方式通过中间的关联来获得远层的关联。以官方例子来看：

```php
class Country extends Model
{
    public function posts()
    {
        return $this->hasManyThrough(
            'App\Post',
            'App\User',
            'country_id', // 用户表外键...
            'user_id', // 文章表外键...
            'id', // 国家表本地键...
            'id' // 用户表本地键...
        );
    }
}
```
可以看到，远程一对多的参数比较多。第一个参数 `related` 是最终被关联的模型，`through` 是中间模型，`firstKey` 是中间模型关于父模型的外键，`secondKey` 是最终被关联的模型关于中间模型的外键，`localKey` 是父模型的主键，`secondLocalKey` 是中间模型的主键：

```php
public function hasManyThrough($related, $through, $firstKey = null, $secondKey = null, $localKey = null, $secondLocalKey = null)
{
    $through = new $through;

    $firstKey = $firstKey ?: $this->getForeignKey();

    $secondKey = $secondKey ?: $through->getForeignKey();

    $localKey = $localKey ?: $this->getKeyName();

    $secondLocalKey = $secondLocalKey ?: $through->getKeyName();

    $instance = $this->newRelatedInstance($related);

    return new HasManyThrough($instance->newQuery(), $this, $through, $firstKey, $secondKey, $localKey, $secondLocalKey);
}

```
HasManyThrough 的初始化：

```php
public function __construct(Builder $query, Model $farParent, Model $throughParent, $firstKey, $secondKey, $localKey, $secondLocalKey)
{
    $this->localKey = $localKey;
    $this->firstKey = $firstKey;
    $this->secondKey = $secondKey;
    $this->farParent = $farParent;
    $this->throughParent = $throughParent;
    $this->secondLocalKey = $secondLocalKey;

    parent::__construct($query, $throughParent);
}

```

`hasManyThrough` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/HasManyThrough.png)

限制条件：

```php
public function addConstraints()
{
    $localValue = $this->farParent[$this->localKey];

    $this->performJoin();

    if (static::$constraints) {
        $this->query->where($this->getQualifiedFirstKeyName(), '=', $localValue);
    }
}

protected function performJoin(Builder $query = null)
{
    $query = $query ?: $this->query;

    $farKey = $this->getQualifiedFarKeyName();

    $query->join($this->throughParent->getTable(), $this->getQualifiedParentKeyName(), '=', $farKey);

    if ($this->throughParentSoftDeletes()) {
        $query->whereNull($this->throughParent->getQualifiedDeletedAtColumn());
    }
}

public function getQualifiedParentKeyName()
{
    return $this->parent->getTable().'.'.$this->secondLocalKey;
}

public function getQualifiedFarKeyName()
{
    return $this->getQualifiedForeignKeyName();
}

public function getQualifiedForeignKeyName()
{
    return $this->related->getTable().'.'.$this->secondKey;
}

public function getQualifiedFirstKeyName()
{
    return $this->throughParent->getTable().'.'.$this->firstKey;
}
```

本例中的限制条件：

```php

select post join user on user.id = post.user_id

select post where user.delete_at is null

select post where user.country_id = 1 (country.id)

```

## morphOne/morphMany 多态关联

多态关联允许我们应用一个表来单独作为多个表的属性，多态关联存在一对一、一对多、多对多的情形。所谓一对一、一对多是指，一个模型只拥有一个属性或多个属性，例如官网中的例子：

> 用户可以「评论」文章和视频。使用多态关联，您可以用一个 comments 表同时满足这两个使用场景

```php
class Post extends Model
{
    /**
     * 获得此文章的所有评论。
     */
    public function comments()
    {
        return $this->morphMany('App\Comment', 'commentable');
    }
}

class Video extends Model
{
    /**
     * 获得此视频的所有评论。
     */
    public function comments()
    {
        return $this->morphMany('App\Comment', 'commentable');
    }
}

```

这个 `comments` 表就是属性表，当文章和视频只能有一个评论的时候，那么就是一对一多态关联；如果文章和视频可以由多个评论的时候，就是一对多多态关联。

这种属性表一般会有两个固定的字段：`commentable_type` 用于标识该条评论是文章的还是视频的、`commentable_id` 用于记录文章或视频的主键 `id`。

我们可以把多态关联看作普通的一对一、一对多关系，只是外键参数是 `type` 与 `id` 的组合。

`related` 是属性表，也就是这里的 `comments`，`type` 参数是属性表中存储父模型类型的列名(commentable_type)，`id` 参数是属性表中存储父模型主键的列名(commentable_id)，而 `name` 专用于省略 `type` 参数与 `id` 参数，`localKey` 是指父模型的主键。

```php
public function morphOne($related, $name, $type = null, $id = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    list($type, $id) = $this->getMorphs($name, $type, $id);

    $table = $instance->getTable();

    $localKey = $localKey ?: $this->getKeyName();

    return new MorphOne($instance->newQuery(), $this, $table.'.'.$type, $table.'.'.$id, $localKey);
}

public function morphMany($related, $name, $type = null, $id = null, $localKey = null)
{
    $instance = $this->newRelatedInstance($related);

    list($type, $id) = $this->getMorphs($name, $type, $id);

    $table = $instance->getTable();

    $localKey = $localKey ?: $this->getKeyName();

    return new MorphMany($instance->newQuery(), $this, $table.'.'.$type, $table.'.'.$id, $localKey);
}

protected function getMorphs($name, $type, $id)
{
    return [$type ?: $name.'_type', $id ?: $name.'_id'];
}
```
一对一、一对多多态关联主要保存属性表中表示类型的列名，还有需要向该类型列中写入的父模型名称，一般来说，默认会写父模型的类名(`App\Post`、`App\Video`)

```php
public function __construct(Builder $query, Model $parent, $type, $id, $localKey)
{
    $this->morphType = $type;

    $this->morphClass = $parent->getMorphClass();

    parent::__construct($query, $parent, $id, $localKey);
}

public function getMorphClass()
{
    $morphMap = Relation::morphMap();

    if (! empty($morphMap) && in_array(static::class, $morphMap)) {
        return array_search(static::class, $morphMap, true);
    }

    return static::class;
}
```

不过我们也可以自定义写入的值：

```php
Relation::morphMap([
    'posts' => 'App\Post',
    'videos' => 'App\Video',
]);
```

这样，就会把 `App\Post` 换成 `posts`，`App\Video` 换成 `videos`。我们来看看这个 `多态映射表` 函数：
 
```php
public static function morphMap(array $map = null, $merge = true)
{
    $map = static::buildMorphMapFromModels($map);

    if (is_array($map)) {
        static::$morphMap = $merge && static::$morphMap
                        ? array_merge(static::$morphMap, $map) : $map;
    }

    return static::$morphMap;
}

protected static function buildMorphMapFromModels(array $models = null)
{
    if (is_null($models) || Arr::isAssoc($models)) {
        return $models;
    }

    return array_combine(array_map(function ($model) {
        return (new $model)->getTable();
    }, $models), $models);
}
```

可以看到，`buildMorphMapFromModels` 函数将字符串 `App\Post` 转为 `model`，并利用 `array_combine` 转为键。

`morphOne` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/morphOne.png)

`morphMany` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/morphMany.png)

限制条件：

```php
public function addConstraints()
{
    if (static::$constraints) {
        parent::addConstraints();

        $this->query->where($this->morphType, $this->morphClass);
    }
}

public function addConstraints()
{
    if (static::$constraints) {
        $this->query->where($this->foreignKey, '=', $this->getParentKey());

        $this->query->whereNotNull($this->foreignKey);
    }
}

```

本例中的限制条件：

```php

select comments where comment.commentable_id = post.id

select comments where comment.commentable_id is not null

select comments where comment.commentable_type = 'App\Post'

```

## morphTo 反向多态关联

和一对一、一对多的 `belongsTo` 相似，多态关联还可以定义反向关联 `morphTo`:

```php
class Comment extends Model
{
    /**
     * 获得拥有此评论的模型。
     */
    public function commentable()
    {
        return $this->morphTo();
    }
}
```
与 `belongsTo` 类似，`morphTo` 也是利用 `debug_backtrace` 获取关联名称。当前如果正处于预加载状态的时候，`Comment` 一般还没有从数据库获取数据，`$this->{$type}` 是空值，这个时候需要去除预加载来初始化：

```php
public function morphTo($name = null, $type = null, $id = null)
{
    $name = $name ?: $this->guessBelongsToRelation();

    list($type, $id) = $this->getMorphs(
        Str::snake($name), $type, $id
    );

    return empty($class = $this->{$type})
                ? $this->morphEagerTo($name, $type, $id)
                : $this->morphInstanceTo($class, $name, $type, $id);
}

protected function morphEagerTo($name, $type, $id)
{
    return new MorphTo(
        $this->newQuery()->setEagerLoads([]), $this, $id, null, $type, $name
    );
}

protected function morphInstanceTo($target, $name, $type, $id)
{
    $instance = $this->newRelatedInstance(
        static::getActualClassNameForMorph($target)
    );

    return new MorphTo(
        $instance->newQuery(), $this, $id, $instance->getKeyName(), $type, $name
    );
}
```
多态的成员变量 `morphType` 代表属性表的类型列，`morphClass`

`MorphTo` 的成员变量只有一个 `morphType`:

```php
public function __construct(Builder $query, Model $parent, $foreignKey, $ownerKey, $type, $relation)
{
    $this->morphType = $type;

    parent::__construct($query, $parent, $foreignKey, $ownerKey, $relation);
}
```

`morphTo` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/morphTo.png)

限制条件与 `belongsTo` 相同：

```php
public function addConstraints()
{
    if (static::$constraints) {
        $table = $this->related->getTable();

        $this->query->where($table.'.'.$this->ownerKey, '=', $this->child->{$this->foreignKey});
    }
}
```

本例中的限制条件

```php

select post where post.id = comments.commentable_id

```

## 多对多多态关联

除了传统的多态关联，您也可以定义「多对多」的多态关联。例如，Post 模型和 Video 模型可以共享一个多态关联至 Tag 模型。 使用多对多多态关联可以让您在文章和视频中共享唯一的标签列表。

```php
class Post extends Model
{
    /**
     * 获得此文章的所有标签。
     */
    public function tags()
    {
        return $this->morphToMany('App\Tag', 'taggable');
    }
}

```
多对多多态关联与多对多关联的代码类似，不同的是中间表不再是两个父模型的蛇形变量，而是 `name` 的复数，值得注意的是 `foreignPivotKey` 代表中间表中对当前 `post` 或者 `video` 的外键，一般会放在 `taggable_id ` 字段中，`relatedPivotKey` 代表中间表中对属性表 `tag` 的外键 `tag_id`:
 
```php
public function morphToMany($related, $name, $table = null, $foreignPivotKey = null,
                                $relatedPivotKey = null, $parentKey = null,
                                $relatedKey = null, $inverse = false)
{
    $caller = $this->guessBelongsToManyRelation();

    $instance = $this->newRelatedInstance($related);

    $foreignPivotKey = $foreignPivotKey ?: $name.'_id';

    $relatedPivotKey = $relatedPivotKey ?: $instance->getForeignKey();

    $table = $table ?: Str::plural($name);

    return new MorphToMany(
        $instance->newQuery(), $this, $name, $table,
        $foreignPivotKey, $relatedPivotKey, $parentKey ?: $this->getKeyName(),
        $relatedKey ?: $instance->getKeyName(), $caller, $inverse
    );
}
```

`MorphToMany` 的构造函数依然有 `morphType` 与 `morphClass`，`morphType` 标识着当前中间表的记录类型是 `Post`，还是 `videos`，`morphClass` 的值默认值是 `Post` 类或者 `videos` 的全名，正向关联的时候，`inverse` 是 `false`，反向关联的时候, `inverse` 是 `true`。

```php
 public function __construct(Builder $query, Model $parent, $name, $table, $foreignPivotKey,
                                $relatedPivotKey, $parentKey, $relatedKey, $relationName = null, $inverse = false)
{
    $this->inverse = $inverse;
    $this->morphType = $name.'_type';
    $this->morphClass = $inverse ? $query->getModel()->getMorphClass() : $parent->getMorphClass();

    parent::__construct(
        $query, $parent, $table, $foreignPivotKey,
        $relatedPivotKey, $parentKey, $relatedKey, $relationName
    );
}
```
正向关联的时候，`parent` 类是 `Post` 类或者 `videos` 类，反向关联的时候 `related` 是 `Post` 类或者 `videos` 类。

限制条件：
```php
protected function addWhereConstraints()
{
    parent::addWhereConstraints();

    $this->query->where($this->table.'.'.$this->morphType, $this->morphClass);

    return $this;
}

protected function addWhereConstraints()
{
    $this->query->where(
        $this->getQualifiedForeignPivotKeyName(), '=', $this->parent->{$this->parentKey}
    );

    return $this;
}

public function getQualifiedForeignPivotKeyName()
{
    return $this->table.'.'.$this->foreignPivotKey;
}
```
官网中例子限制条件转化为 `sql` (假设 `Post` 的主键为 1) ：

```php
where taggables.taggable_id = 1;

where taggables.taggable_type = 'App\Post'

```

`morphToMany` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/morphToMany.png)

限制条件：

```php
public function addConstraints()
{
    $this->performJoin();

    if (static::$constraints) {
        $this->addWhereConstraints();
    }
}

protected function performJoin($query = null)
{
    $query = $query ?: $this->query;

    $baseTable = $this->related->getTable();

    $key = $baseTable.'.'.$this->relatedKey;

    $query->join($this->table, $key, '=', $this->getQualifiedRelatedPivotKeyName());

    return $this;
}

protected function addWhereConstraints()
{
    parent::addWhereConstraints();

    $this->query->where($this->table.'.'.$this->morphType, $this->morphClass);

    return $this;
}

protected function addWhereConstraints()
{
    $this->query->where(
        $this->getQualifiedForeignPivotKeyName(), '=', $this->parent->{$this->parentKey}
    );

    return $this;
}
```

本例中的限制条件：

```php
select tag join tagable on tagable.tag_id = tag.id

select tags where tagable.tagables_id = post.id

select tags where tagable.tagables_type = 'App\Tag'

```

## 多对多多态反向关联

官方文档例子：

```php
class Tag extends Model
{
    /**
     * 获得此标签下所有的文章。
     */
    public function posts()
    {
        return $this->morphedByMany('App\Post', 'taggable');
    }
}

```

与正向关联相反，`relatedPivotKey ` 代表中间表中对 `related` 表 `post` 或者 `video` 的外键，一般会放在 `taggable_id ` 字段中，`foreignPivotKey` 代表中间表中对当前属性表 `tag` 的外键 `tag_id`：

```php
public function morphedByMany($related, $name, $table = null, $foreignPivotKey = null,
                                  $relatedPivotKey = null, $parentKey = null, $relatedKey = null)
{
    $foreignPivotKey = $foreignPivotKey ?: $this->getForeignKey();

    $relatedPivotKey = $relatedPivotKey ?: $name.'_id';

    return $this->morphToMany(
        $related, $name, $table, $foreignPivotKey,
        $relatedPivotKey, $parentKey, $relatedKey, true
    );
}

```

官网中例子限制条件转化为 `sql` (假设 `Tag` 的主键为 1) ：

```php
where taggables.tag_id = 1;

where taggables.taggable_type = 'App\Post'

```

`morphedByMany` 的模型关系如下：

![](http://owql68l6p.bkt.clouddn.com/morphedByMany.png)

限制条件与 `morphToMany` 一致：

```php
public function addConstraints()
{
    $this->performJoin();

    if (static::$constraints) {
        $this->addWhereConstraints();
    }
}

protected function performJoin($query = null)
{
    $query = $query ?: $this->query;

    $baseTable = $this->related->getTable();

    $key = $baseTable.'.'.$this->relatedKey;

    $query->join($this->table, $key, '=', $this->getQualifiedRelatedPivotKeyName());

    return $this;
}

protected function addWhereConstraints()
{
    parent::addWhereConstraints();

    $this->query->where($this->table.'.'.$this->morphType, $this->morphClass);

    return $this;
}

protected function addWhereConstraints()
{
    $this->query->where(
        $this->getQualifiedForeignPivotKeyName(), '=', $this->parent->{$this->parentKey}
    );

    return $this;
}

```

本例中的限制条件

```php
select post join post on post.id = tagables.tagable_id

select post where tagables.tag_id = tag.id

select post where tagables.tagable_type = 'App\Post'

```