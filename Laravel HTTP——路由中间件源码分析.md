title: Laravel Http——路由中间件源码分析
date: 2017-07-27 09:32:14
---

---
# 前言
当进行了路由匹配与路由参数绑定后，接下来就要进行路由闭包或者控制器的运行，在此之前，本文先介绍中间件的相关源码。

# 中间件的搜集
由于定义的中间件方式很灵活，所以在运行控制器或者路由闭包之前，我们需要先将在各个地方注册的所有中间件都搜集到一起，然后集中排序。

```php
public function dispatchToRoute(Request $request)
{
    $route = $this->findRoute($request);

    $request->setRouteResolver(function () use ($route) {
        return $route;
    });

    $this->events->dispatch(new Events\RouteMatched($route, $request));

    $response = $this->runRouteWithinStack($route, $request);

    return $this->prepareResponse($request, $response);
}

protected function runRouteWithinStack(Route $route, Request $request)
{
    $shouldSkipMiddleware = $this->container->bound('middleware.disable') &&
                            $this->container->make('middleware.disable') === true;

    $middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route);

    return (new Pipeline($this->container))
                    ->send($request)
                    ->through($middleware)
                    ->then(function ($request) use ($route) {
                        return $this->prepareResponse(
                            $request, $route->run()
                        );
                    });
}

public function gatherRouteMiddleware(Route $route)
{
    $middleware = collect($route->gatherMiddleware())->map(function ($name) {
        return (array) MiddlewareNameResolver::resolve($name, $this->middleware, $this->middlewareGroups);
    })->flatten();

    return $this->sortMiddleware($middleware);
}
```

路由的中间件大致有两个大的来源：

- 在路由的定义过程中，利用关键字 `middleware` 为路由添加中间件，这种中间件都是在文件 `App\Http\Kernel` 中`$middlewareGroups` 、`$routeMiddleware` 这两个数组定义的中间件别名。
- 在路由控制器的构造函数中，添加中间件，可以在这里定义一个闭包作为中间件，也可以利用中间件别名。

```php
public function gatherMiddleware()
{
    if (! is_null($this->computedMiddleware)) {
        return $this->computedMiddleware;
    }

    $this->computedMiddleware = [];

    return $this->computedMiddleware = array_unique(array_merge(
        $this->middleware(), $this->controllerMiddleware()
    ), SORT_REGULAR);
}
```

### 路由定义的中间件是从 `action` 数组中取出来的：

```php
 public function middleware($middleware = null)
{
    if (is_null($middleware)) {
        return (array) Arr::get($this->action, 'middleware', []);
    }

    if (is_string($middleware)) {
        $middleware = func_get_args();
    }

    $this->action['middleware'] = array_merge(
        (array) Arr::get($this->action, 'middleware', []), $middleware
    );

    return $this;
}
```
### 控制器定义的中间件：

```php
public function controllerMiddleware()
{
    if (! $this->isControllerAction()) {
        return [];
    }

    return ControllerDispatcher::getMiddleware(
        $this->getController(), $this->getControllerMethod()
    );
}

public function getController()
{
    $class = $this->parseControllerCallback()[0];

    if (! $this->controller) {
        $this->controller = $this->container->make($class);
    }

    return $this->controller;
}

protected function getControllerMethod()
{
    return $this->parseControllerCallback()[1];
}

protected function parseControllerCallback()
{
    return Str::parseCallback($this->action['uses']);
}

public static function parseCallback($callback, $default = null)
{
    return static::contains($callback, '@') ? explode('@', $callback, 2) : [$callback, $default];
}
```

当前的路由如果使用控制器的时候，就要解析属性 `use`，解析出控制器的类名与类方法。接下来就需要 `ControllerDispatcher` 类。

在讲解 `ControllerDispatcher` 类之前，我们需要先了解一下控制器中间件：

```php
abstract class Controller
{
    public function middleware($middleware, array $options = [])
    {
        foreach ((array) $middleware as $m) {
            $this->middleware[] = [
                'middleware' => $m,
                'options' => &$options,
            ];
        }

        return new ControllerMiddlewareOptions($options);
    }
}

class ControllerMiddlewareOptions
{
    protected $options;

    public function __construct(array &$options)
    {
        $this->options = &$options;
    }
    
    public function only($methods)
    {
        $this->options['only'] = is_array($methods) ? $methods : func_get_args();

        return $this;
    }

    public function except($methods)
    {
        $this->options['except'] = is_array($methods) ? $methods : func_get_args();

        return $this;
    }
}
```
在为控制器定义中间的是，可以为中间件利用 `only` 指定在当前控制器中调用该中间件的特定控制器方法，也可以利用 `except`指定在当前控制器禁止调用中间件的方法。这些信息都保存在控制器的变量 `middleware` 的 `options` 中。

在搜集控制器的中间件时，就要利用中间件的这些信息：

```php
class ControllerDispatcher
{
	public static function getMiddleware($controller, $method)
    {
        if (! method_exists($controller, 'getMiddleware')) {
            return [];
        }

        return collect($controller->getMiddleware())->reject(function ($data) use ($method) {
            return static::methodExcludedByOptions($method, $data['options']);
        })->pluck('middleware')->all();
    }
    
    protected static function methodExcludedByOptions($method, array $options)
    {
        return (isset($options['only']) && ! in_array($method, (array) $options['only'])) ||
            (! empty($options['except']) && in_array($method, (array) $options['except']));
    }
}
```

在 `ControllerDispatcher` 类中，利用了 `reject` 函数对每一个中间件都进行了控制器方法的判断，排除了不支持该控制器方法的中间件。`pluck` 函数获取了控制器 `$this->middleware[]` 数组中 `middleware` 的所有元素。

# 中间件的解析
中间件解析主要的工作是将路由中中间件的别名转化为中间件全程，主要流程为：

```php
class MiddlewareNameResolver
{
    public static function resolve($name, $map, $middlewareGroups)
    {
        if ($name instanceof Closure) {
            return $name;
        } elseif (isset($map[$name]) && $map[$name] instanceof Closure) {
            return $map[$name];

        } elseif (isset($middlewareGroups[$name])) {
            return static::parseMiddlewareGroup(
                $name, $map, $middlewareGroups
            );

        } else {
            list($name, $parameters) = array_pad(explode(':', $name, 2), 2, null);

            return (isset($map[$name]) ? $map[$name] : $name).
                   (! is_null($parameters) ? ':'.$parameters : '');
        }
    }
}
```

可以看出，解析的中间件对象有三种：闭包、中间件别名、中间件组。

- 对于闭包来说，`resolve` 直接返回闭包；
- 对于中间件别名来说，例如 `auth` ，会从 `App\Http\Kernel` 文件 `$routeMiddleware` 数组中寻找中间件全名 `\Illuminate\Auth\Middleware\Authenticate::class`
- 对于具有参数的中间件别名来说，例如 `throttle:60,1`,会将别名转化为全名 `\Illuminate\Routing\Middleware\ThrottleRequests::60,1`
- 对于中间件组来说，会调用 `parseMiddlewareGroup` 函数。

```php
protected static function parseMiddlewareGroup($name, $map, $middlewareGroups)
{
    $results = [];

    foreach ($middlewareGroups[$name] as $middleware) {
        if (isset($middlewareGroups[$middleware])) {
            $results = array_merge($results, static::parseMiddlewareGroup(
                $middleware, $map, $middlewareGroups
            ));

            continue;
        }

        list($middleware, $parameters) = array_pad(
            explode(':', $middleware, 2), 2, null
        );

        if (isset($map[$middleware])) {
            $middleware = $map[$middleware];
        }

        $results[] = $middleware.($parameters ? ':'.$parameters : '');
    }

    return $results;
}
```
可以看出，对于中间件组来说，就要从 `App\Http\Kernel` 文件 `$$middlewareGroups` 数组中寻找组内的多个中间件，例如中间件组 `api` ：

```php
'api' => [
    'throttle:60,1',
    'bindings',
]
```
解析出的中间件可能存在参数，别名转化为全名后函数返回。值得注意的是，中间件组内不一定都是别名，也有可能是中间件组的组名，例如：

```php
'api' => [
    'throttle:60,1',
    'web',
]

'web' => [
    \App\Http\Middleware\EncryptCookies::class,
    \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
],
```
这时，就需要迭代解析。

# 中间件的排序

```php
public function gatherRouteMiddleware(Route $route)
{
    $middleware = collect($route->gatherMiddleware())->map(function ($name) {
        return (array) MiddlewareNameResolver::resolve($name, $this->middleware, $this->middlewareGroups);
    })->flatten();

    return $this->sortMiddleware($middleware);
}
```

将所有中间件搜集并解析完毕后，接下来就要对中间件的调用顺序做一些调整，以确保中间件功能正常。

```php
protected $middlewarePriority = [
    \Illuminate\Session\Middleware\StartSession::class,
    \Illuminate\View\Middleware\ShareErrorsFromSession::class,
    \Illuminate\Auth\Middleware\Authenticate::class,
    \Illuminate\Session\Middleware\AuthenticateSession::class,
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
    \Illuminate\Auth\Middleware\Authorize::class,
];
```
数组 `middlewarePriority` 中保存着必须有一定顺序的中间件，例如 `StartSession` 中间件就必须运行在 `ShareErrorsFromSession` 之前。因此一旦路由中有这两个中间件，那么就要确保两者的顺序一致。

中间件的排序由函数 `sortMiddleware` 负责：

```php
class SortedMiddleware extends Collection
{
    public function __construct(array $priorityMap, $middlewares)
    {
        if ($middlewares instanceof Collection) {
            $middlewares = $middlewares->all();
        }

        $this->items = $this->sortMiddleware($priorityMap, $middlewares);
    }

    protected function sortMiddleware($priorityMap, $middlewares)
    {
        $lastIndex = 0;

        foreach ($middlewares as $index => $middleware) {
            if (! is_string($middleware)) {
                continue;
            }

            $stripped = head(explode(':', $middleware));

            if (in_array($stripped, $priorityMap)) {
                $priorityIndex = array_search($stripped, $priorityMap);

                if (isset($lastPriorityIndex) && $priorityIndex < $lastPriorityIndex) {
                    return $this->sortMiddleware(
                        $priorityMap, array_values(
                            $this->moveMiddleware($middlewares, $index, $lastIndex)
                        )
                    );
                } else {
                    $lastIndex = $index;
                    $lastPriorityIndex = $priorityIndex;
                }
            }
        }

        return array_values(array_unique($middlewares, SORT_REGULAR));
    }
    
    protected function moveMiddleware($middlewares, $from, $to)
    {
        array_splice($middlewares, $to, 0, $middlewares[$from]);

        unset($middlewares[$from + 1]);

        return $middlewares;
    }
}
```

函数的方法很简单，检测当前中间件数组，查看是否存在中间件是数组 `middlewarePriority` 内元素。如果发现了两个中间件不符合顺序，那么就要调换中间件顺序，然后进行迭代。