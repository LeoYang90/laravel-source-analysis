# 前言

当路由与请求进行正则匹配后，各个路由的参数就获得了它们各自的数值。然而，有些路由参数变量，我们还想要把它转化为特定的对象，这时候就需要中间件的帮助。 `SubstituteBindings` 中间件就是一个将路由参数转化为特定对象的组件，它默认可以将特定名称的路由参数转化数据库模型对象，可以转化已绑定的路由参数为把绑定的对象。

# `SubstituteBindings` 中间件的使用

## 数据库模型隐性转化

首先我们定义了一个带有路由参数的路由：

```php
Route::put('user/{userid}', 'UserController@update');
```

然后我们在路由的控制器方法中或者路由闭包函数中定义一个数据库模型类型的参数，这个参数名与路由参数相同：

```php
class UserController extends Controller
{
    public function update(UserModel $userid)
    {
        $userid->name = 'taylor';
        $userid->update();
    }
}
```

这时，路由的参数会被中间件隐性地转化为 `UserModel`，且模型变量 `$userid` 的主键值为参数变量 `{userid}` 正则匹配后的数值。

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

    $this->assertEquals(1, $values[0]->value);
}


class RouteTestAnotherControllerWithParameterStub extends Controller
{
    public function callAction($method, $parameters)
    {
        $_SERVER['__test.controller_callAction_parameters'] = $parameters;
    }

    public function withModels(RoutingTestUserModel $user)
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

# `SubstituteBindings` 中间件的源码解析

```php
class SubstituteBindings
{
    public function handle($request, Closure $next)
    {
        $this->router->substituteBindings($route = $request->route());

        $this->router->substituteImplicitBindings($route);

        return $next($request);
    }
}
```
从代码来看，`substituteBindings` 用于显示的参数转化，`substituteImplicitBindings` 用于隐性的参数转化。

## 隐性参数转化源码解析

进行隐性参数转化，其步骤为：

- 扫描控制器方法或者闭包函数所有的参数，提取出数据库模型类型对象
- 根据模型类型对象的 `name`，找出与模型对象命名相同的路由参数
- 根据模型类型对象的 `classname`，构建数据库模型类型对象，根据路由参数的数值在数据库中执行 `sql` 语句查询

```php
public function substituteImplicitBindings($route)
{
    ImplicitRouteBinding::resolveForRoute($this->container, $route);
}

class ImplicitRouteBinding
{
    public static function resolveForRoute($container, $route)
    {
        $parameters = $route->parameters();

        foreach ($route->signatureParameters(Model::class) as $parameter) {
            $class = $parameter->getClass();

            if (array_key_exists($parameter->name, $parameters) &&
                ! $route->parameter($parameter->name) instanceof Model) {
                $method = $parameter->isDefaultValueAvailable() ? 'first' : 'firstOrFail';

                $model = $container->make($class->name);

                $route->setParameter(
                    $parameter->name, $model->where(
                        $model->getRouteKeyName(), $parameters[$parameter->name]
                    )->{$method}()
                );
            }
        }
    }
}
```

值得注意的是，显示参数转化的优先级要高于隐性转化，如果当前参数已经被 `model` 函数显示转化，那么该参数并不会进行隐性转化，也就是上面语句 `! $route->parameter($parameter->name) instanceof Model` 的作用。

其中扫描控制器方法参数的功能主要利用反射机制：

```php
public function signatureParameters($subClass = null)
{
    return RouteSignatureParameters::fromAction($this->action, $subClass);
}

class RouteSignatureParameters
{
    public static function fromAction(array $action, $subClass = null)
    {
        $parameters = is_string($action['uses'])
                        ? static::fromClassMethodString($action['uses'])
                        : (new ReflectionFunction($action['uses']))->getParameters();

        return is_null($subClass) ? $parameters : array_filter($parameters, function ($p) use ($subClass) {
            return $p->getClass() && $p->getClass()->isSubclassOf($subClass);
        });
    }
    
    protected static function fromClassMethodString($uses)
    {
        list($class, $method) = Str::parseCallback($uses);

        return (new ReflectionMethod($class, $method))->getParameters();
    }
}
```

## bind 显示参数绑定

路由的 `bind` 功能由专门的 `binders` 数组负责，这个数组中保存着所有的需要显示转化的路由参数与他们的转化闭包函数：

```php
public function bind($key, $binder)
{
    $this->binders[str_replace('-', '_', $key)] = RouteBinding::forCallback(
        $this->container, $binder
    );
}

class RouteBinding
{
    public static function forCallback($container, $binder)
    {
        if (is_string($binder)) {
            return static::createClassBinding($container, $binder);
        }

        return $binder;
    }
    
    protected static function createClassBinding($container, $binding)
    {
        return function ($value, $route) use ($container, $binding) {
            list($class, $method) = Str::parseCallback($binding, 'bind');

            $callable = [$container->make($class), $method];

            return call_user_func($callable, $value, $route);
        };
    }
}
```

可以看出，`bind` 函数可以绑定闭包、`classname@method`、`classname`，如果仅仅绑定了一个类名，那么程序默认调用类中 `bind` 方法。

## model 显示参数绑定

`model` 调用 `bind` 函数，赋给 `bind` 函数一个提前包装好的闭包函数：

```php
public function model($key, $class, Closure $callback = null)
{
    $this->bind($key, RouteBinding::forModel($this->container, $class, $callback));
}

class RouteBinding
{
    public static function forModel($container, $class, $callback = null)
    {
        return function ($value) use ($container, $class, $callback) {
            if (is_null($value)) {
                return;
            }

            $instance = $container->make($class);

            if ($model = $instance->where($instance->getRouteKeyName(), $value)->first()) {
                return $model;
            }

            if ($callback instanceof Closure) {
                return call_user_func($callback, $value);
            }

            throw (new ModelNotFoundException)->setModel($class);
        };
    }
}
```

可以看出，这个闭包函数与隐性转化很相似，都是首先创建数据库模型对象，再利用路由参数值来查询数据库，返回对象。 `model` 还可以提供默认的闭包函数，以供查询不到数据库时调用。

## 显示路由参数转化

当运行中间件 `SubstituteBindings` 时，就会将先前绑定的各个闭包函数执行，并对路由参数进行转化：

```php
public function substituteBindings($route)
{
    foreach ($route->parameters() as $key => $value) {
        if (isset($this->binders[$key])) {
            $route->setParameter($key, $this->performBinding($key, $value, $route));
        }
    }

    return $route;
}

protected function performBinding($key, $value, $route)
{
    return call_user_func($this->binders[$key], $value, $route);
}

public function setParameter($name, $value)
{
    $this->parameters();

    $this->parameters[$name] = $value;
}
```