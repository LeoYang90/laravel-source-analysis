title: Laravel HTTP——控制器方法的参数构建与运行
date: 2017-08-01 16:05:49
---

---
# 前言

经过前面一系列中间件的工作，现在请求终于要达到了正确的控制器方法了。本篇文章主要讲 `laravel` 如何调用控制器方法，并且为控制器方法依赖注入构建参数的过程。

# 路由控制器的调用

我们前面已经解析过中间件的搜集与排序、pipeline 的原理，接下来就要进行路由的 `run` 运行函数：

```php
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
```

路由的 `run` 函数主要负责路由控制器方法与路由闭包函数的运行：

```php
public function run()
{
    $this->container = $this->container ?: new Container;

    try {
        if ($this->isControllerAction()) {
            return $this->runController();
        }

        return $this->runCallable();
    } catch (HttpResponseException $e) {
        return $e->getResponse();
    }
}
```

路由的运行主要靠 `ControllerDispatcher` 这个类：

```php
class Route
{
    protected function isControllerAction()
    {
        return is_string($this->action['uses']);
    }

    protected function runController()
    {
        return (new ControllerDispatcher($this->container))->dispatch(
            $this, $this->getController(), $this->getControllerMethod()
        );
    }
}

class ControllerDispatcher
{
	use RouteDependencyResolverTrait;
	
	public function dispatch(Route $route, $controller, $method)
    {
        $parameters = $this->resolveClassMethodDependencies(
            $route->parametersWithoutNulls(), $controller, $method
        );

        if (method_exists($controller, 'callAction')) {
            return $controller->callAction($method, $parameters);
        }

        return $controller->{$method}(...array_values($parameters));
    }
}
```
上面可以很清晰地看出，控制器的运行分为两步：解析函数参数、调用callAction

## 解析控制器方法参数

解析参数的功能主要由 `ControllerDispatcher` 类的 `RouteDependencyResolverTrait` 这一 `trait` 负责：

```php
trait RouteDependencyResolverTrait
{
    protected function resolveClassMethodDependencies(array $parameters, $instance, $method)
    {
        if (! method_exists($instance, $method)) {
            return $parameters;
        }

        return $this->resolveMethodDependencies(
            $parameters, new ReflectionMethod($instance, $method)
        );
    }
    
    public function resolveMethodDependencies(array $parameters, ReflectionFunctionAbstract $reflector)
    {
        $instanceCount = 0;

        $values = array_values($parameters);

        foreach ($reflector->getParameters() as $key => $parameter) {
            $instance = $this->transformDependency(
                $parameter, $parameters
            );

            if (! is_null($instance)) {
                $instanceCount++;

                $this->spliceIntoParameters($parameters, $key, $instance);
            } elseif (! isset($values[$key - $instanceCount]) &&
                      $parameter->isDefaultValueAvailable()) {
                $this->spliceIntoParameters($parameters, $key, $parameter->getDefaultValue());
            }
        }

        return $parameters;
    }
}
``` 

控制器方法函数参数构造难点在于，参数来源有三种：

- 路由参数赋值
- Ioc 容器自动注入
- 函数自带默认值

在 Ioc 容器自动注入的时候，要保证路由的现有参数中没有相应的类，防止依赖注入覆盖路由绑定的参数：

```php
protected function transformDependency(ReflectionParameter $parameter, $parameters)
{
    $class = $parameter->getClass();

    if ($class && ! $this->alreadyInParameters($class->name, $parameters)) {
        return $this->container->make($class->name);
    }
}

protected function alreadyInParameters($class, array $parameters)
{
    return ! is_null(Arr::first($parameters, function ($value) use ($class) {
        return $value instanceof $class;
	}));
}
```
由 Ioc 容器构造出的参数需要插入到原有的路由参数数组中：

```php
if (! is_null($instance)) {
    $instanceCount++;

    $this->spliceIntoParameters($parameters, $key, $instance);
}

protected function spliceIntoParameters(array &$parameters, $offset, $value)
{
    array_splice(
        $parameters, $offset, 0, [$value]
    );
}
```

当路由的参数数组与 Ioc 容器构造的参数数量不足以覆盖控制器参数个数时，就要去判断控制器是否具有默认参数：

```php
elseif (! isset($values[$key - $instanceCount]) &&
       $parameter->isDefaultValueAvailable()) {
    $this->spliceIntoParameters($parameters, $key, $parameter->getDefaultValue());
}
```

## 调用控制器方法 callAction

所有的控制器并非是直接调用相应方法的，而是通过 `callAction` 函数再分配，如果实在没有相应方法还会调用魔术方法 `__call()`:

```php
public function callAction($method, $parameters)
{
    return call_user_func_array([$this, $method], $parameters);
}

public function __call($method, $parameters)
{
    throw new BadMethodCallException("Method [{$method}] does not exist.");
}
```

# 路由闭包函数的调用

路由闭包函数的调用与控制器方法一样，仍然需要依赖注入，参数构造：

```php
protected function runCallable()
{
    $callable = $this->action['uses'];

    return $callable(...array_values($this->resolveMethodDependencies(
        $this->parametersWithoutNulls(), new ReflectionFunction($this->action['uses'])
    )));
}
```