## paginate 分页

`laravel` 的分页用起来非常简单，只需要对 `query` 调用 `paginate` 函数，把返回的对象扔给前端 `blade` 文件，在 `blade` 文件调用函数 `render` 函数或者 `link` 函数，就可以得到 `上一页`、`下一页` 等等分页特效。

实际上，我们可以简单地把分页服务看作一个前端资源，`render` 函数或者 `link` 函数的结果就是分页前端代码。

如果你还对 `laravel` 的分页不是很熟悉，请先阅读官方文档 ： [分页](https://d.laravel-china.org/docs/5.5/pagination)。

## 分页服务的启动

分页功能也是由一个服务提供者所启动的，`PaginationServiceProvider` 就是负责注册和启动分页服务的服务提供者：

```php
class PaginationServiceProvider extends ServiceProvider
{
    public function register()
    {
        Paginator::viewFactoryResolver(function () {
            return $this->app['view'];
        });

        Paginator::currentPathResolver(function () {
            return $this->app['request']->url();
        });

        Paginator::currentPageResolver(function ($pageName = 'page') {
            $page = $this->app['request']->input($pageName);

            if (filter_var($page, FILTER_VALIDATE_INT) !== false && (int) $page >= 1) {
                return $page;
            }

            return 1;
        });
    }
}
```
我们看到，服务提供者的注册函数为 `Paginator` 设置三个闭包函数：

- viewFactoryResolver 为 `Paginator` 设置了生成前端资源的类，用于获取分页前端代码。
- currentPathResolver 为 `Paginator` 设置了 `url` 的地址。我们知道， `上一页`、`下一页` 等等都是可以执行翻页的操作，所以实际上这些按钮必然含有链接，而链接的地址就是当前请求的 `url` 地址，不同的按钮的链接地址只是 `page` 的参数不同而已。
- currentPageResolver 为 `Paginator` 获取了当前的页数。

```php
public function boot()
{
    $this->loadViewsFrom(__DIR__.'/resources/views', 'pagination');

    if ($this->app->runningInConsole()) {
        $this->publishes([
            __DIR__.'/resources/views' => $this->app->resourcePath('views/vendor/pagination'),
        ], 'laravel-pagination');
    }
}

protected function loadViewsFrom($path, $namespace)
{
    if (is_dir($appPath = $this->app->resourcePath().'/views/vendor/'.$namespace)) {
        $this->app['view']->addNamespace($namespace, $appPath);
    }

    $this->app['view']->addNamespace($namespace, $path);
}
```
服务的启动函数为分页服务设置了默认的前端分页资源。

## 分页服务 paginator

分页服务 `paginator` 函数用于 `queryBuilder`，用于获取分页的数据库数据：

```php
public function paginate($perPage = 15, $columns = ['*'], $pageName = 'page', $page = null)
{
    $page = $page ?: Paginator::resolveCurrentPage($pageName);

    $total = $this->getCountForPagination($columns);

    $results = $total
        ? $this->forPage($page, $perPage)->get($columns) : collect();

    return $this->paginator($results, $total, $perPage, $page, [
        'path' => Paginator::resolveCurrentPath(),
        'pageName' => $pageName,
    ]);
}

protected function paginator($items, $total, $perPage, $currentPage, $options)
{
    return Container::getInstance()->makeWith(LengthAwarePaginator::class, compact(
        'items', 'total', 'perPage', 'currentPage', 'options'
    ));
}
```
也就是说，当我们写下这样的代码时：

```php
DB::table('user')->select('*')->where('status',1)->paginator();
```
我们可以获取到一个 `LengthAwarePaginator` 类对象，对这个对象调用 `render` 函数就可以获取分页前端资源。

我们先来研究一下 `paginator` 函数。

### 获取当前页

我们可以看到，在这个函数中程序先获取当前页数：

```php
public static function resolveCurrentPage($pageName = 'page', $default = 1)
{
    if (isset(static::$currentPageResolver)) {
        return call_user_func(static::$currentPageResolver, $pageName);
    }

    return $default;
}
```
currentPageResolver 就是上一节中 `currentPageResolver` 设置的闭包函数，这个闭包函数从请求参数中获取当前页：

```php
$page = $this->app['request']->input($pageName);
```

### 获取数据库总记录数

计算数据库符合搜索条件的总记录数理所当然的是使用聚合函数 `count` ：

```php
public function getCountForPagination($columns = ['*'])
{
    $results = $this->runPaginationCountQuery($columns);

    if (isset($this->groups)) {
        return count($results);
    } elseif (! isset($results[0])) {
        return 0;
    } elseif (is_object($results[0])) {
        return (int) $results[0]->aggregate;
    } else {
        return (int) array_change_key_case((array) $results[0])['aggregate'];
    }
}

protected function runPaginationCountQuery($columns = ['*'])
{
    return $this->cloneWithout(['columns', 'orders', 'limit', 'offset'])
                ->cloneWithoutBindings(['select', 'order'])
                ->setAggregate('count', $this->withoutSelectAliases($columns))
                ->get()->all();
}
```

### 获取当前页数据

获取当前页当然是使用 `forPage` 函数：

```php
$results = $total
        ? $this->forPage($page, $perPage)->get($columns) : collect();
```

### 初始化 LengthAwarePaginator

`paginator` 函数利用 Ioc 容器来生成 `LengthAwarePaginator` 实例：

```php
protected function paginator($items, $total, $perPage, $currentPage, $options)
{
    return Container::getInstance()->makeWith(LengthAwarePaginator::class, compact(
        'items', 'total', 'perPage', 'currentPage', 'options'
    ));
}
```
`LengthAwarePaginator` 的初始化：

```php
public function __construct($items, $total, $perPage, $currentPage = null, array $options = [])
{
    foreach ($options as $key => $value) {
        $this->{$key} = $value;
    }

    $this->total = $total;
    $this->perPage = $perPage;
    $this->lastPage = max((int) ceil($total / $perPage), 1);
    $this->path = $this->path !== '/' ? rtrim($this->path, '/') : $this->path;
    $this->currentPage = $this->setCurrentPage($currentPage, $this->pageName);
    $this->items = $items instanceof Collection ? $items : Collection::make($items);
}
```

## 分页资源 render

对 `LengthAwarePaginator` 调用 `render` 函数会得到分页所需要的前端资源：

```php
public function render($view = null, $data = [])
{
    return new HtmlString(static::viewFactory()->make($view ?: static::$defaultView, array_merge($data, [
        'paginator' => $this,
        'elements' => $this->elements(),
    ]))->render());
}

```
当我们使用默认的分页样式的时候，不需要向 `render` 函数传入 `view` 参数，此时程序会自动加载默认的前端资源:

```php
public static $defaultView = 'pagination::default';
```
该资源的默认地址是 `illuminate\Pagination\resources\views\default.blade.php`:

```php
@if ($paginator->hasPages())
    <ul class="pagination">
        {{-- Previous Page Link --}}
        @if ($paginator->onFirstPage())
            <li class="disabled"><span><<</span></li>
        @else
            <li><a href="{{ $paginator->previousPageUrl() }}" rel="prev"><<</a></li>
        @endif

        {{-- Pagination Elements --}}
        @foreach ($elements as $element)
            {{-- "Three Dots" Separator --}}
            @if (is_string($element))
                <li class="disabled"><span>{{ $element }}</span></li>
            @endif

            {{-- Array Of Links --}}
            @if (is_array($element))
                @foreach ($element as $page => $url)
                    @if ($page == $paginator->currentPage())
                        <li class="active"><span>{{ $page }}</span></li>
                    @else
                        <li><a href="{{ $url }}">{{ $page }}</a></li>
                    @endif
                @endforeach
            @endif
        @endforeach

        {{-- Next Page Link --}}
        @if ($paginator->hasMorePages())
            <li><a href="{{ $paginator->nextPageUrl() }}" rel="next">>></a></li>
        @else
            <li class="disabled"><span>>></span></li>
        @endif
    </ul>
@endif
```
可以看到，分页效果的代码分为三部分：前一页、后一页、分页元素。

### 前一页

如果当前页是第一页的话，`前一页` 按钮需要置灰：

```php
public function onFirstPage()
{
    return $this->currentPage() <= 1;
}
```
否则的话，就要为 `前一页` 按钮赋予链接：

```php
public function previousPageUrl()
{
    if ($this->currentPage() > 1) {
        return $this->url($this->currentPage() - 1);
    }
}

public function url($page)
{
    if ($page <= 0) {
        $page = 1;
    }

    $parameters = [$this->pageName => $page];

    if (count($this->query) > 0) {
        $parameters = array_merge($this->query, $parameters);
    }

    return $this->path
                    .(Str::contains($this->path, '?') ? '&' : '?')
                    .http_build_query($parameters, '', '&')
                    .$this->buildFragment();
}
```

如果列表页中存在一些搜索条件，这些搜索条件会被加载到 `$this->query` 成员变量中，生成 `url` 的时候，这些搜索添加会被加到 `request` 的参数中。可以使用 `append` 方法附加查询参数到分页链接中：

```php
public function appends($key, $value = null)
{
    if (is_array($key)) {
        return $this->appendArray($key);
    }

    return $this->addQuery($key, $value);
}

protected function appendArray(array $keys)
{
    foreach ($keys as $key => $value) {
        $this->addQuery($key, $value);
    }

    return $this;
}
```
 
### 下一页

与 `前一页` 类似，如果已经在最后一页，那么 `下一页` 按钮将会被置灰：

```php
public function hasMorePages()
{
    return $this->currentPage() < $this->lastPage();
}
```
下一页的链接：

```php
public function nextPageUrl()
{
    if ($this->lastPage() > $this->currentPage()) {
        return $this->url($this->currentPage() + 1);
    }
}
```
`上一页` 与 `下一页` 按钮的功能比较简单，至于中间的分页特效比较复杂，我们由下一节来说。

## 分页 elements

我们先说一下不同的分页样式：

- 当我们设置两侧页数为 3 时，当前数据总页数小于 8 页时分页效果：

![](http://owql68l6p.bkt.clouddn.com/QQ20171001-104935@2x.png)

- 总页数大于 6 页，且当前页在前 8 页（2 * 3 + 2）时分页效果：

![img](http://owql68l6p.bkt.clouddn.com/QQ20171001-102833@2x.png)

- 当前页在前 6 页与后 6 页之间分页效果：

![img](http://owql68l6p.bkt.clouddn.com/QQ20171001-102858@2x.png)

- 当前页在最后 6 页时分页效果：

![img](http://owql68l6p.bkt.clouddn.com/QQ20171001-102916@2x.png)

分页效果样式的关键来源于 `UrlWindow`，这个类用于根据总页数与当前页的不同来控制不同的分页样式。

```php
protected function elements()
{
    $window = UrlWindow::make($this);

    return array_filter([
        $window['first'],
        is_array($window['slider']) ? '...' : null,
        $window['slider'],
        is_array($window['last']) ? '...' : null,
        $window['last'],
    ]);
}

public static function make(PaginatorContract $paginator, $onEachSide = 3)
{
    return (new static($paginator))->get($onEachSide);
}

public function get($onEachSide = 3)
{
    if ($this->paginator->lastPage() < ($onEachSide * 2) + 6) {
        return $this->getSmallSlider();
    }

    return $this->getUrlSlider($onEachSide);
}
```

### 小型分页 getSmallSlider

如果当前总页数小于 `($onEachSide * 2) + 6` 的话，就会调用小型分页效果，这种小型分页效果直接将所有页数全部显示：

```php
protected function getSmallSlider()
{
    return [
        'first'  => $this->paginator->getUrlRange(1, $this->lastPage()),
        'slider' => null,
        'last'   => null,
    ];
}

public function getUrlRange($start, $end)
{
    return collect(range($start, $end))->mapWithKeys(function ($page) {
        return [$page => $this->url($page)];
    })->all();
}
```

### CloseToBeginning 分页效果

当前页数位于前 `($onEachSide * 2)` 页时：

```php
protected function getUrlSlider($onEachSide)
{
    $window = $onEachSide * 2;

    if (! $this->hasPages()) {
        return ['first' => null, 'slider' => null, 'last' => null];
    }

    if ($this->currentPage() <= $window) {
        return $this->getSliderTooCloseToBeginning($window);
    }

    elseif ($this->currentPage() > ($this->lastPage() - $window)) {
        return $this->getSliderTooCloseToEnding($window);
    }

    return $this->getFullSlider($onEachSide);
}

protected function getSliderTooCloseToBeginning($window)
{
    return [
        'first' => $this->paginator->getUrlRange(1, $window + 2),
        'slider' => null,
        'last' => $this->getFinish(),
    ];
}

public function getFinish()
{
    return $this->paginator->getUrlRange(
        $this->lastPage() - 1,
        $this->lastPage()
    );
}
```

假设我们设置当前两侧页数为 3，当前页为 5，总页数22，函数 `getSliderTooCloseToBeginning` 返回结果为：

```php
return [
    'first' => [
        1 => '/www.example.com/example?page=1', 
        2 => '/www.example.com/example?page=2'
        3 => '/www.example.com/example?page=3'
        4 => '/www.example.com/example?page=4'
        5 => '/www.example.com/example?page=5'
        6 => '/www.example.com/example?page=6'
        7 => '/www.example.com/example?page=7'
        8 => '/www.example.com/example?page=8'],
    'slider' => null,
    'last' => [
        21 => '/www.example.com/example?page=21',
        22 => '/www.example.com/example?page=22'],
];
```
这个时候 `element` 函数返回数据：

```php
protected function elements()
{
    $window = UrlWindow::make($this);

    return array_filter([
        $window['first'],
        is_array($window['slider']) ? '...' : null,
        $window['slider'],
        is_array($window['last']) ? '...' : null,
        $window['last'],
    ]);
}

//返回结果
[
    [
        1 => '/www.example.com/example?page=1', 
        2 => '/www.example.com/example?page=2',
        3 => '/www.example.com/example?page=3',
        4 => '/www.example.com/example?page=4',
        5 => '/www.example.com/example?page=5',
        6 => '/www.example.com/example?page=6',
        7 => '/www.example.com/example?page=7',
        8 => '/www.example.com/example?page=8',
    ],                                        //$window['first']
    ‘...’,                                    //is_array($window['last']) ? '...' : null
    [
        21 => '/www.example.com/example?page=21',
        22 => '/www.example.com/example?page=22',
    ],                                        //$window['last']
]
```

### TooCloseToEnding 分页效果

当前页数位于后 `($onEachSide * 2)` 页时：

```php
protected function getSliderTooCloseToEnding($window)
{
    $last = $this->paginator->getUrlRange(
        $this->lastPage() - ($window + 2),
        $this->lastPage()
    );

    return [
        'first' => $this->getStart(),
        'slider' => null,
        'last' => $last,
    ];
}

public function getStart()
{
    return $this->paginator->getUrlRange(1, 2);
}
```
假设我们设置当前两侧页数为 3，当前页为 18，总页数22，函数 `getSliderTooCloseToEnding` 返回结果为：

```php
return [
    'first' => [
        1 => '/www.example.com/example?page=1', 
        2 => '/www.example.com/example?page=2'
    ],
    'slider' => null,
    'last' => [
        15 => '/www.example.com/example?page=15',
        16 => '/www.example.com/example?page=16',
        17 => '/www.example.com/example?page=17',
        18 => '/www.example.com/example?page=18',
        19 => '/www.example.com/example?page=19',
        20 => '/www.example.com/example?page=20',
        21 => '/www.example.com/example?page=21',
        22 => '/www.example.com/example?page=22',
    ],
];
```

这个时候 `element` 函数返回数据：

```php
[
	[
	    1 => '/www.example.com/example?page=1', 
	    2 => '/www.example.com/example?page=2'
   ],
   '...',
   [
   		15 => '/www.example.com/example?page=15',
   		16 => '/www.example.com/example?page=16',
   		17 => '/www.example.com/example?page=17',
   		18 => '/www.example.com/example?page=18',
   		19 => '/www.example.com/example?page=19',
   		20 => '/www.example.com/example?page=20',
   		21 => '/www.example.com/example?page=21',
   		22 => '/www.example.com/example?page=22',
   ]
]
```

### FullSlider 分页效果

当前页数位于中间时：

```php
protected function getFullSlider($onEachSide)
{
    return [
        'first'  => $this->getStart(),
        'slider' => $this->getAdjacentUrlRange($onEachSide),
        'last'   => $this->getFinish(),
    ];
}

public function getAdjacentUrlRange($onEachSide)
{
    return $this->paginator->getUrlRange(
        $this->currentPage() - $onEachSide,
        $this->currentPage() + $onEachSide
    );
}
```

假设我们设置当前两侧页数为 3，当前页为 10，总页数22，函数 `getFullSlider` 返回结果为：

```php
return [
    'first' => [
        1 => '/www.example.com/example?page=1', 
        2 => '/www.example.com/example?page=2'
    ],
    'slider' => [
        7 => '/www.example.com/example?page=7', 
        8 => '/www.example.com/example?page=8', 
        9 => '/www.example.com/example?page=9',
        10 => '/www.example.com/example?page=10', 
        11 => '/www.example.com/example?page=11', 
        12 => '/www.example.com/example?page=12',
        13 => '/www.example.com/example?page=13', 
    ],
    'last' => [
        21 => '/www.example.com/example?page=21',
        22 => '/www.example.com/example?page=22',
    ],
];
```

这个时候 `element` 函数返回数据：

```php
[
	[
	    1 => '/www.example.com/example?page=1', 
	    2 => '/www.example.com/example?page=2'
   ],
   '...',
   [
       7 => '/www.example.com/example?page=7', 
       8 => '/www.example.com/example?page=8', 
       9 => '/www.example.com/example?page=9',
       10 => '/www.example.com/example?page=10', 
       11 => '/www.example.com/example?page=11', 
       12 => '/www.example.com/example?page=12',
       13 => '/www.example.com/example?page=13',
   ],
   '...',
   [
   		21 => '/www.example.com/example?page=21',
   		22 => '/www.example.com/example?page=22',
   ]
]
```

## simplePaginate 简单分页

简单分页相比以上的功能来说，精简了 `elements` 的特效：

```php
public function simplePaginate($perPage = 15, $columns = ['*'], $pageName = 'page', $page = null)
{
    $page = $page ?: Paginator::resolveCurrentPage($pageName);

    $this->skip(($page - 1) * $perPage)->take($perPage + 1);

    return $this->simplePaginator($this->get($columns), $perPage, $page, [
        'path' => Paginator::resolveCurrentPath(),
        'pageName' => $pageName,
    ]);
}

protected function simplePaginator($items, $perPage, $currentPage, $options)
{
    return Container::getInstance()->makeWith(Paginator::class, compact(
        'items', 'perPage', 'currentPage', 'options'
    ));
}
```

分页服务的类不再使用 `LengthAwarePaginator` 类，而开始使用 `Paginator`，这两个类最大的不同在于 `render` 函数：

```php
public static $defaultSimpleView = 'pagination::simple-default';
 
public function render($view = null, $data = [])
{
    return new HtmlString(
        static::viewFactory()->make($view ?: static::$defaultSimpleView, array_merge($data, [
            'paginator' => $this,
        ]))->render()
    );
}
```
`render` 函数调用的前端资源默认地址为 `illuminate\Pagination\resources\views\simple-default.blade.php`:

```php
@if ($paginator->hasPages())
    <ul class="pagination">
        {{-- Previous Page Link --}}
        @if ($paginator->onFirstPage())
            <li class="disabled"><span>@lang('pagination.previous')</span></li>
        @else
            <li><a href="{{ $paginator->previousPageUrl() }}" rel="prev">@lang('pagination.previous')</a></li>
        @endif

        {{-- Next Page Link --}}
        @if ($paginator->hasMorePages())
            <li><a href="{{ $paginator->nextPageUrl() }}" rel="next">@lang('pagination.next')</a></li>
        @else
            <li class="disabled"><span>@lang('pagination.next')</span></li>
        @endif
    </ul>
@endif

```

可以看到，简单分页只有 `上一页`、`下一页` 两个按钮。