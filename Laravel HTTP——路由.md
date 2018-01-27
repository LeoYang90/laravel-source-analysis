# 前言
作为一个 web 后台框架，路由无疑是极其重要的一部分。本博客接下来几篇文章都将会围绕路由这一主题来展开讨论，分别讲述：
> - 路由的使用
> - 路由属性注册
> - 路由的正则编译与匹配
> - 路由的中间件
> - 路由的控制器与参数绑定
> - RESTful 路由

和之前一样，第一篇将会利用单元测试样例说明我们在平时可能用到的 route 的 api 函数用法，后面几篇文章将会剖析 laravel 的 route 源码。下面开始介绍 laravel 中路由的各种用法。

---
# 路由属性注册
所有 Laravel 路由都定义在位于 routes 目录下的路由文件中，这些文件通过框架自动加载。routes/web.php 文件定义了 web 界面的路由，这些路由被分配了 web 中间件组，从而可以提供 session 和 csrf 防护等功能。routes/api.php 中的路由是无状态的，被分配了 api 中间件组。

对大多数应用而言，都是从 routes/web.php 文件开始定义路由。
## 路由 method 方法
我们可以注册路由来响应任何 HTTP 请求：

```php
Route::get($uri, $callback);
Route::post($uri, $callback);
Route::put($uri, $callback);
Route::patch($uri, $callback);
Route::delete($uri, $callback);
Route::options($uri, $callback);
```
有时候还需要注册路由响应多个 HTTP 请求——这可以通过 `match` 方法来实现。或者，可以使用 `any` 方法注册一个路由来响应所有 HTTP 请求：

```php
Route::match(['get', 'post'], '/', function () {
    //
});

Route::any('foo', function () {
    //
});
```
值得注意的是，一般的HTML表单仅仅支持`get`、`post`，并不支持`put`、`patch`、`delete`等动作，这时候就需要在前端添加一个隐藏的 `_method` 字段到给表单中，其值被用作 `HTTP` 请求方法名：

```php
<input type="hidden" name="_method" value="PUT">
```
在 web 路由文件中所有请求方式为`PUT`、`POST`或`DELETE`的 HTML 表单都会包含一个`CSRF`令牌字段，否则，请求会被拒绝。关于 `CSRF` 的更多细节，可以参考 [浅谈CSRF攻击方式](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html?login=1)：

```php
<form method="POST" action="/profile">
    {{ csrf_field() }}
    ...
</form>
```

## 路由 scheme 协议
对于 web 后台框架来说，路由的 `scheme` 底层协议一般使用 `http`、`https`:

```php
Route::get('foo/{bar}', ['http', function () {
    }]);
Route::get('foo/{bar}', ['https', function () {
    }]);
```
## 路由 domain 子域名
子域名可以像 URI 一样被分配给路由参数，子域名可以通过路由属性中的 `domain` 来指定：

```php
Route::domain('api.name.bar')
     ->get('foo/bar', function ($name) {
        return $name;
    });
    
Route::get('foo/bar', ['domain' => 'api.name.bar', function ($name) {
        return $name;
    }]);
```

##  路由 prefix 前缀
可以为路由添加一个给定 `URI` 前缀，通过利用路由属性的 `prefix` 指定：

```php
Route::prefix('pre')
     ->get('foo/bar', function () {
    });
    
Route::get('foo/bar', ['prefix' => 'pre', function () {
    }]);
    
Route::get('foo/bar', function () {
    })->prefix('pre');

```

## 路由 where 正则约束
可以为路由的 `URI` 参数指定正则约束：

```php
Route::get('{one}', ['where' => ['one' => '(.+)'], function () {
    }]);
    
Route::get('{one}', function () {
    })->where('one', '(.+)');
```

如果想要路由参数在全局范围内被给定正则表达式约束，可以使用 `pattern` 方法。在 `RouteServiceProvider` 类的 `boot` 方法中定义约束模式：

```php
public function boot()
{
    Route::pattern('one', '(.+)');
    parent::boot();
}
```
## 路由 middleware 中间件
为路由添加中间件，通过利用路由属性的 `middleware` 指定：

```php
Route::middleware('web')
     ->get('foo/bar', function () {
    });

Route::get('foo/bar', ['middleware' => 'web', function () {
    }]);

Route::get('foo/bar', function () {
    })->middleware('web');
```
## 路由 namespace 属性
可以为路由的控制器添加 `namespace` 来指定控制器的命名空间：

```php
Route::namespace('Namespace\\Example\\')
     ->get('foo/bar', function () {
    });
    
Route::get('foo/bar', ['namespace' => 'Namespace\\Example\\', function () {
    }]);
```
## 路由 uses 属性
可以为路由添加 `URI` 对应的执行逻辑，例如闭包或者控制器：

```php
Route::get('foo/bar', ['uses' => function () {
    }]);
    
Route::get('foo/bar', ['uses' => ‘Illuminate\Tests\Routing\RouteTestControllerStub@index’]);

Route::get('foo/bar')->uses(function () {
    });
    
Route::get('foo/bar')->uses(‘Illuminate\Tests\Routing\RouteTestControllerStub@index’);

```
## 路由 as 别名 
可以为路由指定别名，通过路由属性的 `as` 来指定：

```php
Route::as('Foo')
     ->get('foo/bar', function () {
    });
    
Route::name('Foo')
     ->get('foo/bar', function () {
    });

Route::get('foo/bar', ['as' => 'Foo', function () {
    }]);

Route::get('foo/bar', function () {
    })->name('Foo');
```
## 路由 group 群组属性
可以为一系列具有类似属性的路由归为同一组，利用 `group` 将这些路由归并到一起：

```php
Route::group(['domain'     => 'group.domain.name',
              'prefix'     => 'grouppre',
              'where'      => ['one' => '(.+)'], 
              'middleware' => 'groupMiddleware',
              'namespace'  => 'Namespace\\Group\\', 
              'as'         => 'Group::',]
              function () {
                  Route::get('/replace',‘domain’ => 'route.domain.name'，
                                        'uses'   => function () {
                                        return 'replace';
                  });

                 Route::get('additional/{one}/{two}', 'prefix'     => 'routepre',
                                                      'where'      => '['one' => '([0-9]+)','two' => '(.+)']',
                                                      'middleware' => 'routeMiddleware',
                                                      'namespace'  => 'Namespace\\Group\\', 
                                                      'as'         => 'Route',
                                                      'use         => 'function () {
                                                      return 'additional';
                 });
});

$this->assertEquals('replace', $router->dispatch(Request::create('http://route.domain.name/grouppre/replace', 'GET'))->getContent());

$this->assertEquals('additional', $router->dispatch(Request::create('http://group.domain.name/routepre/grouppre/additional/111/add', 'GET'))->getContent());

$routes = $router->getRoutes()->getRoutes();
$action = $routes[0]->getAction();
$this->assertEquals('Namespace\\Group\\', $action['namespace']);
$this->assertEquals('Group::', $action['as']);

$routes = $router->getRoutes()->getRoutes();
$action = $routes[1]->getAction();
$this->assertEquals(['groupMiddleware', 'routeMiddleware'], $action['middleware']);
$this->assertEquals('Namespace\\Group\\Namespace\\Group\\', $action['namespace']);
$this->assertEquals('Group::Route', $action['as']);
```

`group` 群组的属性分为两类：替换型、递增型。当群组属性与路由属性重复的时候，替换型属性会用路由的属性替换群组的属性，递增型的属性会综合路由和群组的属性。

在上面的例子可以看出：

- `domain` 这个属性是替换型属性，路由的属性会覆盖和替换群组的这几个属性；
- `prefix`、`middleware`、`namespace`、`as` 、`where` 这几个属性是递增型属性，路由的属性和群组属性会相互结合。

另外值得注意的是：

- 路由的 `prefix` 属性具有优先级,因此上面第二个路由的 `uri` 是 `routepre/grouppre/additional/111/add`,而不是 `grouppre/routepre/additional/111/add`；
- `where`属性对于相同的路由参数会替换，不同的路由参数会结合，因此上面 `where` 中 `one` 被替换，`two` 被结合进来

# 路由参数与匹配
laravel 允许在注册定义路由的时候设定路由参数，以供控制器或者闭包所用。路由参数可以设定在 `URI` 中，也可以设定在 `domain` 中。
## 路由编码匹配

对于已编码的请求 `URI`，框架会自动进行解码然后进行匹配:

```php
$router = $this->getRouter();
 $router->get('foo/bar/åαф', function () {
     return 'hello';
});
$this->assertEquals('hello', $router->dispatch(Request::create('foo/bar/%C3%A5%CE%B1%D1%84', 'GET'))->getContent());

$router = $this->getRouter();
$route = $router->get('foo/{file}', function ($file) {
    return $file;
});
$this->assertEquals('oxygen%20', $router->dispatch(Request::create('http://test.com/foo/oxygen%2520', 'GET'))->getContent());
```

## 路由参数

路由参数总是通过花括号进行包裹，这些参数在路由被执行时会被传递到路由的闭包。路由参数不能包含 - 字符，需要的话可以使用 _ 替代。

```php
$router = $this->getRouter();
$route = $router->get('foo/{age}', ['domain' => 'api.{name}.bar', function ($name, $age) {
   return $name.$age;
}]);
$this->assertEquals('taylor25', $router->dispatch(Request::create('http://api.taylor.bar/foo/25', 'GET'))->getContent());

$route = new Route('GET', 'images/{id}.{ext}', function () {
    });

$request1 = Request::create('images/1.png', 'GET');
$this->assertTrue($route->matches($request1));
$route->bind($request1);
$this->assertTrue($route->hasParameter('id'));
$this->assertFalse($route->hasParameter('foo'));
$this->assertEquals('1', $route->parameter('id'));
$this->assertEquals('png', $route->parameter('ext'));        

```

## 路由可选参数
有时候可能需要指定可选的路由参数，这可以通过在参数名后加一个 ? 标记来实现，这种情况下需要给相应的变量指定默认值：

```php
$router = $this->getRouter();
$router->get('{foo?}/{baz?}', function ($name = 'taylor', $age = 25) {
     return $name.$age;
});
$this->assertEquals('fred25', $router->dispatch(Request::create('fred', 'GET'))->getContent());

$router->get('default/{foo?}/{baz?}', function ($name, $age = 25) {
     return $name.$age;
})->default('name', 'taylor');
$this->assertEquals('fred25', $router->dispatch(Request::create('fred', 'GET'))->getContent());
```

## 路由参数正则约束
可以使用路由实例上的 `where` 方法来约束路由参数的格式。`where` 方法接收参数名和一个正则表达式来定义该参数如何被约束：

```php
Route::get('user/{name}', function ($name) {
    //
})->where('name', '[A-Za-z]+');
```
如果想要路由参数在全局范围内被给定正则表达式约束，可以使用 `pattern` 方法。在 `RouteServiceProvider` 类的 `boot` 方法中定义约束模式:

```php
public function boot()
{
    Route::pattern('id', '[0-9]+');
    parent::boot();
}
```
值得注意的是，路由参数是不允许出现 `/` 字符的，例如：

```php
$router->get('{one?}', [
            'uses' => function ($one = null){
                 return $one;
            },
        ]);
$request = Request::create('foo/bar/baz', 'GET');
$this->assertFalse($route->matches($request2));
```
上例中 `one` 只能匹配 `foo`，不能匹配 `foo/bar/baz`,这时就需要对 `one` 进行正则约束：

```php
public function testLeadingParamDoesntReceiveForwardSlashOnEmptyPath()
{
    $router = $this->getRouter();
    $router->get('{one?}', [
            'uses' => function ($one = null){
                 return $one;
            },
            'where' => ['one' => '(.+)'],
        ]);
        
    $this->assertEquals('foo', $router->dispatch(Request::create('/foo', 'GET'))->getContent());
    $this->assertEquals('foo/bar/baz', $router->dispatch(Request::create('/foo/bar/baz', 'GET'))->getContent());
}
```

# 路由中间件
HTTP 中间件为过滤进入应用的 HTTP 请求提供了一套便利的机制。例如，Laravel 内置了一个中间件来验证用户是否经过认证，如果用户没有经过认证，中间件会将用户重定向到登录页面，否则如果用户经过认证，中间件就会允许请求继续往前进入下一步操作。

Laravel框架自带了一些中间件，包括认证、CSRF 保护中间件等等。所有的中间件都位于 app/Http/Middleware 目录。

## 中间件之前/之后/终止
一个中间件是请求前还是请求后执行取决于中间件本身。比如，以下中间件会在请求处理前执行一些任务：

```php
class BeforeMiddleware
{
    public function handle($request, Closure $next)
    {
        // 执行动作

        return $next($request);
    }
}

class AfterMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        // 执行动作

        return $response;
    }
}
```
有时候中间件可能需要在 HTTP 响应发送到浏览器之后做一些工作。比如，Laravel 内置的“session”中间件会在响应发送到浏览器之后将 Session 数据写到存储器中，为了实现这个功能，需要定义一个终止中间件并添加 terminate 方法到这个中间件：

```php
class StartSession
{
    public function handle($request, Closure $next)
    {
        return $next($request);
    }

    public function terminate($request, $response)
    {
        // 存储session数据...
    }
}
```
## 全局中间件
如果你想要中间件在每一个 HTTP 请求期间被执行，只需要将相应的中间件类设置到 `app/Http/Kernel.php` 的数组属性 `$middleware` 中即可。

```php
protected $middleware = [
        \Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];
```

## 路由中间件
如果你想要分配中间件到指定路由，可以传递完整的类名：

```php
use App\Http\Middleware\CheckAge;

Route::get('admin/profile', function () {
    //
})->middleware(CheckAge::class);
```
或者可以给中间件提供一个别名：

```php
public function testDefinedClosureMiddleware()
 {
    $router = $this->getRouter();
    $router->get('foo/bar', ['middleware' => 'foo', function () {
        return 'hello';
    }]);
    $router->aliasMiddleware('foo', function ($request, $next) {
        return 'caught';
    });
    $this->assertEquals('caught', $router->dispatch(Request::create('foo/bar', 'GET'))->getContent());
 }
```

也可以应该在 `app/Http/Kernel.php` 文件中分配给该中间件一个 `key`，默认情况下，该类的 `$routeMiddleware` 属性包含了 `Laravel` 自带的中间件，要添加你自己的中间件，只需要将其追加到后面并为其分配一个 `key`，例如：

```php
protected $routeMiddleware = [
    'auth' => \Illuminate\Auth\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class,
    'can' => \Illuminate\Auth\Middleware\Authorize::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
];

Route::get('admin/profile', function () {
    //
})->middleware('auth');
```
使用数组分配多个中间件到路由：

```php
Route::get('/', function () {
    //
})->middleware('first', 'second');
```
## 中间件组
有时候你可能想要通过指定一个键名的方式将相关中间件分到同一个组里面，从而更方便将其分配到路由中，这可以通过使用 `HTTP Kernel` 的 `$middlewareGroups` 属性实现。

`Laravel` 自带了开箱即用的 `web` 和 `api` 两个中间件组以分别包含可以应用到 `Web UI` 和 `API` 路由的通用中间件：

```php
protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\VerifyCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],

    'api' => [
        'throttle:60,1',
        'auth:api',
    ],
];

Route::get('/', function () {
    //
})->middleware('web');
```
值得注意的是，中间件组中可以循环嵌套中间件组：

```php
 public function testMiddlewareGroupsCanReferenceOtherGroups()
 {
    unset($_SERVER['__middleware.group']);
    $router = $this->getRouter();
    $router->get('foo/bar', ['middleware' => 'web', function () {
        return 'hello';
    }]);

    $router->aliasMiddleware('two', 'Illuminate\Tests\Routing\RoutingTestMiddlewareGroupTwo');
    $router->middlewareGroup('first', ['two:abigail']);
    $router->middlewareGroup('web', ['Illuminate\Tests\Routing\RoutingTestMiddlewareGroupOne', 'first']);

    $this->assertEquals('caught abigail', $router->dispatch(Request::create('foo/bar', 'GET'))->getContent());
    $this->assertTrue($_SERVER['__middleware.group']);

    unset($_SERVER['__middleware.group']);
}
```

## 中间件参数
中间件还可以接收额外的自定义参数，例如，如果应用需要在执行给定动作之前验证认证用户是否拥有指定的角色，可以创建一个 `CheckRole` 来接收角色名作为额外参数。

额外的中间件参数会在 `$next` 参数之后传入中间件：

```php
namespace App\Http\Middleware;

use Closure;

class CheckRole
{
    public function handle($request, Closure $next, $role)
    {
        if (! $request->user()->hasRole($role)) {
            // Redirect...
        }

        return $next($request);
    }

}

Route::put('post/{id}', function ($id) {
    //
})->middleware('role:editor');
```
## 中间件的顺序

当 `router` 中有多个中间件的时候，中间件的执行顺序并不是严格按照中间件数组进行的，框架中存在一个数组 `$middlewarePriority`,规定了这个数组中各个中间件的顺序：

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
当我们使用了上面其中多个中间件的时候，框架会自动按照上面的数组进行排序：

```php
 public function testMiddlewarePrioritySorting()
{
    $middleware = [
        Placeholder1::class,
        SubstituteBindings::class,
        Placeholder2::class,
        Authenticate::class,
        Placeholder3::class,
    ];

    $router = $this->getRouter();

    $router->middlewarePriority = [Authenticate::class, SubstituteBindings::class, Authorize::class];

    $route = $router->get('foo', ['middleware' => $middleware, 'uses' => function ($name) {
        return $name;
    }]);

    $this->assertEquals([
        Placeholder1::class,
        Authenticate::class,
        SubstituteBindings::class,
        Placeholder2::class,
        Placeholder3::class,
    ], $router->gatherRouteMiddleware($route));
}
```
# 控制器
## 控制器类
更普遍的方法是使用控制器来组织管理这些行为。控制器可以将相关的 `HTTP` 请求封装到一个类中进行处理。通常控制器存放在 `app/Http/Controllers` 目录中.

所有的 Laravel 控制器应该继承自 Laravel 自带的控制器基类 `Controller`，控制器基类提供了一些很方便的方法如 `middleware`，用于添加中间件到控制器动作：

```php
class UserController extends Controller
{
    public function show($id)
    {
        return view('user.profile', ['user' => User::findOrFail($id)]);
    }
}

Route::get('user/{id}', 'UserController@show');
```

## 单动作控制器
如果想要定义一个只处理一个动作的控制器，可以在这个控制器中定义 __invoke 方法,当为这个单动作控制器注册路由的时候，不需要指定方法：

```php
public function testDispatchingCallableActionClasses()
{
    $router = $this->getRouter();
    $router->get('foo/bar', 'Illuminate\Tests\Routing\ActionStub');

    $this->assertEquals('hello', $router->dispatch(Request::create('foo/bar', 'GET'))->getContent());

    $router->get('foo/bar2', [
        'uses' => 'Illuminate\Tests\Routing\ActionStub@func',
    ]);

    $this->assertEquals('hello2', $router->dispatch(Request::create('foo/bar2', 'GET'))->getContent());
}

class ActionStub extends Controller
{    
    public function __invoke()
    {
        return 'hello';
    }
}
```
## 控制器中间件
将中间件放在控制器构造函数中更方便，在控制器的构造函数中使用 middleware 方法你可以很轻松的分配中间件给该控制器。你甚至可以限定该中间件应用到该控制器类的指定方法：

```php
class UserController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('log')->only('index');
        $this->middleware('subscribed')->except('store');
    }
}
```
## callAction 方法
值得注意的是每次执行控制器方法都会先执行控制器的 `callAction` 函数：

```php
public function callAction($method, $parameters)
{
    return call_user_func_array([$this, $method], $parameters);
}
```
测试样例：

```php

unset($_SERVER['__test.controller_callAction_parameters']);
$router->get(($str = str_random()).'/{one}/{two}', 'Illuminate\Tests\Routing\RouteTestAnotherControllerWithParameterStub@oneArgument');
$router->dispatch(Request::create($str.'/one/two', 'GET'));
 $this->assertEquals(['one' => 'one', 'two' => 'two'], $_SERVER['__test.controller_callAction_parameters']);


class RouteTestAnotherControllerWithParameterStub extends Controller
{
    public function callAction($method, $parameters)
    {
        $_SERVER['__test.controller_callAction_parameters'] = $parameters;
    }

    public function oneArgument($one)
    {
    }
}
```
## __call方法
和普通类一样，若控制器中没有对应 `classname@method` 中的 `method` ,则会调用类的 `__call` 函数。

```php
public function testCallableControllerRouting()
{
    $router = $this->getRouter();

    $router->get('foo/bar', 'Illuminate\Tests\Routing\RouteTestControllerCallableStub@bar');
    $router->get('foo/baz', 'Illuminate\Tests\Routing\RouteTestControllerCallableStub@baz');

    $this->assertEquals('bar', $router->dispatch(Request::create('foo/bar', 'GET'))->getContent());
    $this->assertEquals('baz', $router->dispatch(Request::create('foo/baz', 'GET'))->getContent());
}

class RouteTestControllerCallableStub extends Controller
{
    public function __call($method, $arguments = [])
    {
        return $method;
    }
}
```
# 路由参数依赖注入与绑定
Laravel 使用服务容器解析所有的 Laravel 控制器，因此，可以在控制器的构造函数中类型声明任何依赖，这些依赖会被自动解析并注入到控制器实例中。路由的参数绑定可以分为两种：显示绑定与隐示绑定。

## 路由隐示绑定

- 控制器方法期望输入路由参数，只需要将路由参数放到其他依赖之后

```php
Route::put('user/{id}', 'UserController@update');

class UserController extends Controller
{
    public function update(Request $request, $id)
    {
    }
}
```

- 可以在控制器的动作方法中进行依赖的类型提示，例如，我们可以在某个方法中类型提示 `Illuminate\Http\Request` 实例：

```php
class UserController extends Controller
{
    public function store(Request $request)
    {
        $name = $request->input('name');
    }
}
```

- 可以为控制器的动作方法中添加数据库模型的主键，框架会自动利用主键来获取对应的记录，需要注意的是，route定义路由的路由参数必须和控制器内的变量名相同，例如下例中路由参数 `userid` 和控制器参数 `userid`:

```php
Route::put('user/{userid}', 'UserController@update');

class UserController extends Controller
{
    public function update(UserModel $userid)
    {
        $userid->name = 'taylor';
        $userid->update();
    }
}
```
综合测试样例：

```php
public function testImplicitBindingsWithOptionalParameter()
{
    unset($_SERVER['__test.controller_callAction_parameters']);
    $router->get(($str = str_random()).'/{user}/{defaultNull?}/{team?}', [
        'middleware' => SubstituteBindings::class,
        'uses' => 'Illuminate\Tests\Routing\RouteTestAnotherControllerWithParameterStub@withModels',
    ]);
    
    $router->dispatch(Request::create($str.'/1', 'GET'));

    $values = array_values($_SERVER['__test.controller_callAction_parameters']);

    $this->assertInstanceOf('Illuminate\Http\Request', $values[0]);
    $this->assertEquals(1, $values[1]->value);
    $this->assertNull($values[2]);
    $this->assertInstanceOf('Illuminate\Tests\Routing\RoutingTestTeamModel', $values[3]);
}


class RouteTestAnotherControllerWithParameterStub extends Controller
{
    public function callAction($method, $parameters)
    {
        $_SERVER['__test.controller_callAction_parameters'] = $parameters;
    }

    public function withModels(Request $request, RoutingTestUserModel $user, $defaultNull = null, RoutingTestTeamModel $team = null)
    {
    }
}

class RoutingTestUserModel extends Model
{
    public function getRouteKeyName()
    {
        return 'id';
    }

    public function where($key, $value)
    {
        $this->value = $value;

        return $this;
    }

    public function first()
    {
        return $this;
    }

    public function firstOrFail()
    {
        return $this;
    }
}

class RoutingTestTeamModel extends Model
{
    public function getRouteKeyName()
    {
        return 'id';
    }

    public function where($key, $value)
    {
        $this->value = $value;

        return $this;
    }

    public function first()
    {
        return $this;
    }

    public function firstOrFail()
    {
        return $this;
    }
}

```

## 路由显示绑定

除了隐示地转化路由参数外，我们还可以给路由参数显示提供绑定。显示绑定有 `bind`、`model` 两种方法。

- 通过 `bind` 为参数绑定闭包函数：

```php
public function testRouteBinding()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
         return $name;
    }]);
    $router->bind('bar', function ($value) {
        return strtoupper($value);
    });
    $this->assertEquals('TAYLOR', $router->dispatch(Request::create('foo/taylor', 'GET'))->getContent());
}
```

- 通过 `bind` 为参数绑定类方法，可以指定 `classname@method`，也可以直接使用类名，默认会调用类的 `bind` 函数：

```php
public function testRouteClassBinding()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
    	return $name;
    }]);
    $router->bind('bar', 'Illuminate\Tests\Routing\RouteBindingStub');
    $this->assertEquals('TAYLOR', $router->dispatch(Request::create('foo/taylor', 'GET'))->getContent());
}

public function testRouteClassMethodBinding()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
        return $name;
    }]);
    $router->bind('bar', 'Illuminate\Tests\Routing\RouteBindingStub@find');
    $this->assertEquals('dragon', $router->dispatch(Request::create('foo/Dragon', 'GET'))->getContent());
}

class RouteBindingStub
{
    public function bind($value, $route)
    {
        return strtoupper($value);
    }

    public function find($value, $route)
    {
        return strtolower($value);
    }
}
```

- 通过 `model` 为参数绑定数据库模型，路由的参数就不需要和控制器方法中的变量名相同，`laravel` 会利用路由参数的值去调用 `where` 方法查找对应记录：

```php
if ($model = $instance->where($instance->getRouteKeyName(), $value)->first()) {
     return $model;
}
```
测试样例如下：

```php
public function testModelBinding()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
        return $name;
    }]);
    $router->model('bar', 'Illuminate\Tests\Routing\RouteModelBindingStub');
    $this->assertEquals('TAYLOR', $router->dispatch(Request::create('foo/taylor', 'GET'))->getContent());
}

class RouteModelBindingStub
{
    public function getRouteKeyName()
    {
        return 'id';
    }

    public function where($key, $value)
    {
        $this->value = $value;

        return $this;
    }

    public function first()
    {
        return strtoupper($this->value);
    }
}
```

- 若绑定的 `model` 并没有找到对应路由参数的记录，可以在 `model` 中定义一个闭包函数，路由参数会调用闭包函数：

```php
public function testModelBindingWithCustomNullReturn()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
        return $name;
    }]);
    $router->model('bar', 'Illuminate\Tests\Routing\RouteModelBindingNullStub', function () {
        return 'missing';
    });
    $this->assertEquals('missing', $router->dispatch(Request::create('foo/taylor', 'GET'))->getContent());
}

public function testModelBindingWithBindingClosure()
{
    $router = $this->getRouter();
    $router->get('foo/{bar}', ['middleware' => SubstituteBindings::class, 'uses' => function ($name) {
        return $name;
    }]);
    $router->model('bar', 'Illuminate\Tests\Routing\RouteModelBindingNullStub', function ($value) {
        return (new RouteModelBindingClosureStub())->findAlternate($value);
    });
    $this->assertEquals('tayloralt', $router->dispatch(Request::create('foo/TAYLOR', 'GET'))->getContent());
}

class RouteModelBindingNullStub
{
    public function getRouteKeyName()
    {
        return 'id';
    }

    public function where($key, $value)
    {
        return $this;
    }

    public function first()
    {
    }
}

class RouteModelBindingClosureStub
{
    public function findAlternate($value)
    {
        return strtolower($value).'alt';
    }
}

```

# router扩展方法
router支持添加自定义的方法，只需要利用 `macro` 函数来注册对应的函数名和函数实现：

```php
public function testMacro()
{
    $router = $this->getRouter();
    $router->macro('webhook', function () use ($router) {
        $router->match(['GET', 'POST'], 'webhook', function () {
            return 'OK';
        });
    });
    $router->webhook();
    $this->assertEquals('OK', $router->dispatch(Request::create('webhook', 'GET'))->getContent());
    $this->assertEquals('OK', $router->dispatch(Request::create('webhook', 'POST'))->getContent());
}
```