title: Laravel HTTP——添加路由源码分析
tags:
  - php
  - laravel
  - router
  - 源码
categories:
  - php
author: leo yang
date: 2017-07-19 17:46:00
---
# 前言
作为 `laravel` 极其重要的一部分，`route` 功能贯穿着整个网络请求，是 `request` 生命周期的主干。本文主要讲述 `route` 服务的注册与启动、路由的属性注册。本篇内容相对简单，更多的是框架添加路由的整体设计流程。

# route 服务的注册
`laravel` 在接受到请求后，先进行了服务容器与 `http` 核心的初始化，再进行了请求 `request` 的构造与分发。

`route` 服务的注册—— `RoutingServiceProvider` 发生在服务容器 `container` 的初始化上;

`route` 服务的启动与加载—— `RouteServiceProvider` 发生在 `request` 的分发上。

## route 服务的注册——RoutingServiceProvider

所有需要 `laravel` 服务的请求都会加载入口文件 `index.php`:

```php
require __DIR__.'/../bootstrap/autoload.php';

$app = require_once __DIR__.'/../bootstrap/app.php';
```
第一句我们在之前的博客提过，是实现 `PSR0`	、`PSR4`标准自动加载的功能模块，第二句就是今天说的 `container` 的初始化：

```php
$app = new Illuminate\Foundation\Application(
    realpath(__DIR__.'/../')
);
```
`Application` :

```php
namespace Illuminate\Foundation;

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
    public function __construct($basePath = null)
    {
        if ($basePath) {
            $this->setBasePath($basePath);
        }

        $this->registerBaseBindings();

        $this->registerBaseServiceProviders();

        $this->registerCoreContainerAliases();
    }
}
```

路由服务的注册就在 `registerBaseServiceProviders()` 这个函数中：

```php
protected function registerBaseServiceProviders()
{
    $this->register(new EventServiceProvider($this));

    $this->register(new LogServiceProvider($this));

    $this->register(new RoutingServiceProvider($this));
}
```

`RoutingServiceProvider` :

```php
namespace Illuminate\Routing;

class RoutingServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->registerRouter();

        ...
    }
    
    protected function registerRouter()
    {
        $this->app->singleton('router', function ($app) {
            return new Router($app['events'], $app);
        });
    }
    
    ...
    
}
```

可以看到，`RoutingServiceProvider` 做的事情比较简单，就是向服务容易中注册 `router`。

## `route` 服务的启动与加载——RouteServiceProvider

`laravel` 在初始化 `Application` 后，就要进行 `http/Kernel` 的构造： 

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
``` 

初始化结束后，就会调用 `handle` 函数，这个函数用于 `laravel` 各个功能服务的注册启动，还有 `request` 的分发：

```php
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } 
    
    return $response;
}

protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();//各种服务的注册与启动

    return (new Pipeline($this->app))//请求的分发
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
    }
```

路由服务的启动与加载就在其中一个函数中 `bootstrap`，这个函数用于各种服务的注册与启动，比较复杂，我们有机会在以后单独来说。

总之，这个函数会调用 `RouteServiceProvider` 这个类的两个函数： 注册——`register`、启动——`boot`。

由于 `route` 的注册工作由之前 `RoutingServiceProvider` 完成，所以 `RouteServiceProvider` 的 `register` 是空的，这里它只负责路由的启动与加载工作，我们主要看 `boot`：

```php
namespace Illuminate\Foundation\Support\Providers;

class RouteServiceProvider extends ServiceProvider
{
    public function register()
    {
        //
    }
    
    public function boot()
    {
        $this->setRootControllerNamespace();

        if ($this->app->routesAreCached()) {
            $this->loadCachedRoutes();
        } else {
            $this->loadRoutes();

            $this->app->booted(function () {
                $this->app['router']->getRoutes()->refreshNameLookups();
                $this->app['router']->getRoutes()->refreshActionLookups();
            });
        }
    }
    
    protected function loadCachedRoutes()
    {
        $this->app->booted(function () {
            require $this->app->getCachedRoutesPath();
        });
    }

    protected function loadRoutes()
    {
        if (method_exists($this, 'map')) {
            $this->app->call([$this, 'map']);
        }
    }
}

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
    public function routesAreCached()
    {
        return $this['files']->exists($this->getCachedRoutesPath());
    }

    public function getCachedRoutesPath()
    {
        return $this->bootstrapPath().'/cache/routes.php';
    }
}
```

从 `boot` 中可以看出，`laravel` 首先去寻找路由的缓存文件，没有缓存文件再去进行加载路由。缓存文件一般在 `bootstrap/cache/routes.php` 文件中。

加载路由主要调用 `map` 函数，这个函数一般在 `App\Providers\RouteServiceProvider` 这个类中，这个类继承上面的 `Illuminate\Foundation\Support\Providers\RouteServiceProvider`:

```php
use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;

class RouteServiceProvider extends ServiceProvider
{
    public function map()
    {
        $this->mapApiRoutes();

        $this->mapWebRoutes();

        //
    }
    
    protected function mapWebRoutes()
    {
        Route::middleware('web')
             ->namespace($this->namespace)
             ->group(base_path('routes/web.php'));
    }
    
    protected function mapApiRoutes()
    {
        Route::prefix('api')
             ->middleware('api')
             ->namespace($this->namespace)
             ->group(base_path('routes/api.php'));
    }
}
    
```

`laravle` 将路由分为两个大组：`api`、`web`。这两个部分的路由分别写在两个文件中：`routes/web.php`、`routes/api.php`。

## 路由的加载

所谓的路由加载，就是将定义路由时添加的属性，例如 'name'、'domain'、'scheme' 等等保存起来，以待后用。

- `laravel` 定义路由的属性的方法很灵活，可以定义在路由群组前，例如：

```php
Route::domain('route.domain.name')
     ->group(function() {
			Route::get('foo','controller@method');
     })
```

- 可以定义在路由群组中，例如：

```php
Route::group('domain' => 'group.domain.name',function() {
    		Route::get('foo','controller@method');
    })

```

- 可以定义在 `method` 的前面，例如：

```php
Route::domain('route.domain.name')
     ->get('foo','controller@method');
```

- 可以定义在 `method` 中，例如：

```php
Route::get('foo',['domain' => 'route.domain.name','use' => 'controller@method']);
```

- 还可以定义在 `method` 后，例如：

```php
 Route::get('{one}', 'use' => 'controller@method')
      ->where('one', '(.+)');
```

事实上，路由的加载功能主要有三个类负责： `Illuminate\Routing\Router`、`Illuminate\Routing\Route`、`Illuminate\Routing\RouteRegistrar`。

`Router` 在整个路由功能中都是起着中枢的作用， `RouteRegistrar` 主要负责位于 `group`、`method` 这些函数之前的属性注册，例如上面的第一种和第三种，`route` 主要负责位于 `group`、`method` 这些函数之后的属性注册，例如第五种。

# RouteRegistrar 路由加载
## 属性注册
当我们想要在 `Route` 后面直接利用 `domain()`、`name()` 等函数来为路由注册属性的时候，我们实际调用的是 `router` 的魔术方法 `__call()`:

```php
namespace Illuminate\Routing;

class Router implements RegistrarContract, BindingRegistrar
{
    public function __call($method, $parameters)
    {
        if (static::hasMacro($method)) {
            return $this->macroCall($method, $parameters);
        }

        return (new RouteRegistrar($this))->attribute($method, $parameters[0]);
    }
}
```
在类 `RouteRegistrar` 中：

```php
class RouteRegistrar
{
    protected $allowedAttributes = [
    	'as', 'domain', 'middleware', 'name', 'namespace', 'prefix',
    ];
    
    public function attribute($key, $value)
    {
        if (! in_array($key, $this->allowedAttributes)) {
            throw new InvalidArgumentException("Attribute [{$key}] does not exist.");
        }

        $this->attributes[array_get($this->aliases, $key, $key)] = $value;

        return $this;
    }
}
```
## 添加路由

注册属性之后，创建路由的时候，可以仅仅提供 `uri`，可以提供 `uri` 与 闭包，可以提供 `uri` 与 控制器，可以提供 `uri` 与数组：

```php
Route::as('Foo')
     ->namespace('Namespace\\Example\\')
     ->get('foo/bar');//仅仅 uri
     
Route::as('Foo')
     ->namespace('Namespace\\Example\\')
     ->get('foo/bar', function () {
       }); //uri 与闭包
    
Route::as('Foo')
     ->namespace('Namespace\\Example\\')
     ->get('foo/bar', 'controller@method');//uri 与控制器     
     
Route::as('Foo')
     ->namespace('Namespace\\Example\\')
     ->get('foo/bar', ['as'=> 'foo','use' =>'controller@method']);//uri 与数组 
```


利用 `get`、`post` 等方法创建新的路由时，会调用类 `RouteRegistrar` 中的魔术方法 `__call()`：

```php
class RouteRegistrar
{
    protected $passthru = [
        'get', 'post', 'put', 'patch', 'delete', 'options', 'any',
    ];
    
    public function __call($method, $parameters)
    {
        if (in_array($method, $this->passthru)) {
            return $this->registerRoute($method, ...$parameters);
        }

        if (in_array($method, $this->allowedAttributes)) {
            return $this->attribute($method, $parameters[0]);
        }

        throw new BadMethodCallException("Method [{$method}] does not exist.");
    }
    
    protected function registerRoute($method, $uri, $action = null)
    {
        if (! is_array($action)) {
            $action = array_merge($this->attributes, $action ? ['uses' => $action] : []);
        }

        return $this->router->{$method}($uri, $this->compileAction($action));
    }
    
    protected function compileAction($action)
    {
        if (is_null($action)) {
            return $this->attributes;
        }

        if (is_string($action) || $action instanceof Closure) {
            $action = ['uses' => $action];
        }

        return array_merge($this->attributes, $action);
    }
}
```

也就是说，`RouteRegistrar` 在这里会为闭包或控制器等所有非数组的 `action` 添加 `use` 键，然后才会去 `router` 中创建路由。

## 添加路由群组

注册属性之后，还可以创建路由群组，但是这时路由群组不允许添加属性 `action`：

```php
class RouteRegistrar
{
    public function group($callback)
    {
        $this->router->group($this->attributes, $callback);
    }
}

```

# Router 路由群组加载

路由群组的功能可以不断叠加递归，因此每次调用 `group` ，都要用新路由群组的属性与旧路由群组属性合并，以待新的路由去继承。 `group` 参数可以是闭包函数，也可以是包含定义路由的文件路径。

```php
public function group(array $attributes, $routes)
{
    $this->updateGroupStack($attributes);

    $this->loadRoutes($routes);

    array_pop($this->groupStack);
}

protected function updateGroupStack(array $attributes)
{
    if (! empty($this->groupStack)) {
        $attributes = RouteGroup::merge($attributes, end($this->groupStack));
    }

    $this->groupStack[] = $attributes;
}

protected function loadRoutes($routes)
{
    if ($routes instanceof Closure) {
        $routes($this);
    } else {
        $router = $this;

        require $routes;
    }
}
```
关于路由群组属性的合并, 

- `prefix`、`as`、`namespace` 这几个属性会连接在一起，例如 `prefix1/prefix2/prefix3`，
- `where` 属性数组相同的会被替换，不同的会被合并。
- `domain` 属性会被替换。
-  其他属性，例如 `middleware` 数组会直接被合并，即使存在相同的元素。

```php
class RouteGroup
{
    public static function merge($new, $old)
    {
        if (isset($new['domain'])) {
            unset($old['domain']);
        }

        $new = array_merge(static::formatAs($new, $old), [
            'namespace' => static::formatNamespace($new, $old),
            'prefix' => static::formatPrefix($new, $old),
            'where' => static::formatWhere($new, $old),
        ]);

        return array_merge_recursive(Arr::except(
            $old, ['namespace', 'prefix', 'where', 'as']
        ), $new);
    }
}

```
# Router 路由加载

添加路由需要很多步骤，需要将路由本身的属性和路由群组的属性相结合。

```php
public function get($uri, $action = null)
{
    return $this->addRoute(['GET', 'HEAD'], $uri, $action);
}

protected function addRoute($methods, $uri, $action)
{
    return $this->routes->add($this->createRoute($methods, $uri, $action));
}

protected function createRoute($methods, $uri, $action)
{
      
    if ($this->actionReferencesController($action)) {
        $action = $this->convertToControllerAction($action);
    }

    $route = $this->newRoute(
        $methods, $this->prefix($uri), $action
    );

    if ($this->hasGroupStack()) {
        $this->mergeGroupAttributesIntoRoute($route);
    }

    $this->addWhereClausesToRoute($route);

    return $route;
}
```
从上面来看，添加一个新的路由需要：

- 给路由的控制器添加 `group` 的 `namespace`
- 给路由的 `uri` 添加 `group` 的 `prefix` 前缀
- 创建新的路由
- 更新路由的属性信息
- 为路由添加 `router`-`pattern` 正则约束
- 路由添加到 `RouteCollection` 中

## 控制器 namespace 

路由控制器的命名空间一般不用特别指定，默认值是 `\App\Http\Controllers`，每次创建新的路由，都要将默认的命名空间添加到控制器中去：

```php
protected function actionReferencesController($action)
{
    if (! $action instanceof Closure) {
        return is_string($action) || (isset($action['uses']) && is_string($action['uses']));
    }

    return false;
}

protected function convertToControllerAction($action)
{
    if (is_string($action)) {
        $action = ['uses' => $action];
    }

    if (! empty($this->groupStack)) {
        $action['uses'] = $this->prependGroupNamespace($action['uses']);
    }

    $action['controller'] = $action['uses'];

    return $action;
}

protected function prependGroupNamespace($class)
{
    $group = end($this->groupStack);

    return isset($group['namespace']) && strpos($class, '\\') !== 0
            ? $group['namespace'].'\\'.$class : $class;
}
```

## uri 前缀

在创建新的路由前，需要将路由群组的 `prefix` 添加到路由的 `uri` 中：

```php
protected function prefix($uri)
{
    return trim(trim($this->getLastGroupPrefix(), '/').'/'.trim($uri, '/'), '/') ?: '/';
}

public function getLastGroupPrefix()
{
    if (! empty($this->groupStack)) {
        $last = end($this->groupStack);

        return isset($last['prefix']) ? $last['prefix'] : '';
    }

    return '';
}
```

## 创建新的路由

路由的创建需要 `Route` 类：

```php
protected function newRoute($methods, $uri, $action)
{
    return (new Route($methods, $uri, $action))
                ->setRouter($this)
                ->setContainer($this->container);
}
```
关于 `Router` 类添加新的路由我们在下一部分详细说。

## 更新路由属性信息

创建新的路由之后，需要将路由本身的属性 `action` 与路由群组的属性结合在一起：

```php
public function hasGroupStack()
{
    return ! empty($this->groupStack);
}

protected function mergeGroupAttributesIntoRoute($route)
{
    $route->setAction($this->mergeWithLastGroup($route->getAction()));
}
```

## 添加全局正则约束到路由

上一篇文章我们说过，我们可以为路由通过 `pattern` 方法添加全局的参数正则约束，所有每次添加新的路由都要将这个全局正则约束添加到路由中：

```php
public function pattern($key, $pattern)
{
    $this->patterns[$key] = $pattern;
}
    
protected function addWhereClausesToRoute($route)
{
    $route->where(array_merge(
        $this->patterns, isset($route->getAction()['where']) ? $route->getAction()['where'] : []
    ));

    return $route;
}
```
# Route 路由加载

前面说过，路由的创建是由 `Route` 这个类完成的：

```php
 public function __construct($methods, $uri, $action)
{
    $this->uri = $uri;
    $this->methods = (array) $methods;
    $this->action = $this->parseAction($action);

    if (in_array('GET', $this->methods) && ! in_array('HEAD', $this->methods)) {
        $this->methods[] = 'HEAD';
    }

    if (isset($this->action['prefix'])) {
        $this->prefix($this->action['prefix']);
    }
}
```
由此可以看出，路由的创建主要是路由的各个属性的初始化，其中值得注意的有两个： `action` 与 `prefix`

## action 解析

```php
protected function parseAction($action)
{
    return RouteAction::parse($this->uri, $action);
}
```
我们可以看出，添加新的路由时， `action` 属性需要利用 `RouteAction` 类：

```php
class RouteAction
{
    public static function parse($uri, $action)
    {
        if (is_null($action)) {
            return static::missingAction($uri);
        }

        if (is_callable($action)) {
            return ['uses' => $action];
        }

        elseif (! isset($action['uses'])) {
            $action['uses'] = static::findCallable($action);
        }

        if (is_string($action['uses']) && ! Str::contains($action['uses'], '@')) {
            $action['uses'] = static::makeInvokable($action['uses']);
        }

        return $action;
    }
    
    protected static function findCallable(array $action)
    {
        return Arr::first($action, function ($value, $key) {
            return is_callable($value) && is_numeric($key);
        });
    }
    
    protected static function makeInvokable($action)
    {
        if (! method_exists($action, '__invoke')) {
            throw new UnexpectedValueException("Invalid route action: [{$action}].");
        }

        return $action.'@__invoke';
    }
}

```
前面的博客我们说过，创建路由的时候，除了为路由分配控制器之外，还可以为路由分配闭包函数，还有类函数，例如之前说的单动作控制器：

```php


$router->get('foo/bar2', [‘domain’ => 'www.example.com', 'Illuminate\Tests\Routing\ActionStub']);

class ActionStub
{
    public function __invoke()
    {
        return 'hello';
    }
}
```

因此，解析 `action` 主要做两件事：

- 为闭包函数添加 `use` 键。对于此时没有 `use` 键的路由，由于之前在 `Router` 中已经为控制器添加 `use` 键，因此这时没有 `use` 键的，必然是闭包函数，在这里直接或者在 `action` 中寻找闭包函数后，为闭包函数添加 `use` 键。

- 单动作控制器添加 `__invoke `。对于单动作控制器来说，此时已经和控制器一样拥有 'use' 键，但是并没有 `@` 符号，此时就会调用 `makeInvokable` 函数来将 `__invoke` 添加到后面。

## prefix 前缀

路由自身也有 `prefix` 属性，而且这个属性要加在其他 `prefix` 的最前面,作为路由的 `uri`：

```php
public function prefix($prefix)
{
    $uri = rtrim($prefix, '/').'/'.ltrim($this->uri, '/');

    $this->uri = trim($uri, '/');

    return $this;
}
```

# Route 路由属性加载

除了 `RouteRegistrar` 之外，`Route` 也可以为路由添加属性：

## prefix 前缀

```php
public function prefix($prefix)
{
    $uri = rtrim($prefix, '/').'/'.ltrim($this->uri, '/');

    $this->uri = trim($uri, '/');

    return $this;
}
```

## where 正则约束

```php
public function where($name, $expression = null)
{
    foreach ($this->parseWhere($name, $expression) as $name => $expression) {
        $this->wheres[$name] = $expression;
    }
 	
 	return $this;
}     

protected function parseWhere($name, $expression)
{
    return is_array($name) ? $name : [$name => $expression];
}
```

## middleware 中间件

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

## uses 控制器

```php
public function uses($action)
{
    $action = is_string($action) ? $this->addGroupNamespaceToStringUses($action) : $action;

    return $this->setAction(array_merge($this->action, $this->parseAction([
        'uses' => $action,
        'controller' => $action,
    ])));
}
```

## name 命名

```php
public function name($name)
{
    $this->action['as'] = isset($this->action['as']) ? $this->action['as'].$name : $name;

    return $this;
}
```

# RouteCollection 添加路由

在上面的部分，我们看到添加路由的代码：

```php
protected function addRoute($methods, $uri, $action)
{
    return $this->routes->add($this->createRoute($methods, $uri, $action));
}
```
新创建的路由会加入到 `RouteCollection` 中，会更新类中的 `routes`、`allRoutes`、`nameList`、`actionList`。

```php
public function add(Route $route)
{
    $this->addToCollections($route);

    $this->addLookups($route);

    return $route;
}

protected function addToCollections($route)
{
    $domainAndUri = $route->domain().$route->uri();

    foreach ($route->methods() as $method) {
        $this->routes[$method][$domainAndUri] = $route;
    }

    $this->allRoutes[$method.$domainAndUri] = $route;
}

protected function addLookups($route)
{   
    $action = $route->getAction();

    if (isset($action['as'])) {
        $this->nameList[$action['as']] = $route;
    }

   if (isset($action['controller'])) {
        $this->addToActionList($action, $route);
    }
}

protected function addToActionList($action, $route)
{
    $this->actionList[trim($action['controller'], '\\')] = $route;
}
```

我们在上面路由的注册启动章节说道，路由的启动是 `namespace Illuminate\Foundation\Support\Providers\RouteServiceProvider` 完成的，调用的是 `boot` 函数：

```php
public function boot()
{
    $this->setRootControllerNamespace();

    if ($this->app->routesAreCached()) {
        $this->loadCachedRoutes();
    } else {
        $this->loadRoutes();

        $this->app->booted(function () {
            $this->app['router']->getRoutes()->refreshNameLookups();
        });
    }
}
```

在最后一句，程序将会在所有服务都启动后运行 `refreshNameLookups` 函数，把所有的 `name` 属性加载到 `RouteCollection` 中:

```php
public function refreshNameLookups()
{
    $this->nameList = [];

    foreach ($this->allRoutes as $route) {
        if ($route->getName()) {
            $this->nameList[$route->getName()] = $route;
        }
    }
}
```
测试样例如下：

```php
public function testRouteCollectionCanRefreshNameLookups()
{
    $routeIndex = new Route('GET', 'foo/index', [
        'uses' => 'FooController@index',
    ]);

    $this->assertNull($routeIndex->getName());

    $this->routeCollection->add($routeIndex)->name('route_name');

    $this->assertNull($this->routeCollection->getByName('route_name'));

    $this->routeCollection->refreshNameLookups();
    $this->assertEquals($routeIndex, $this->routeCollection->getByName('route_name'));
}
```