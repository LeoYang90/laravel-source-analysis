# 前言
上一篇文章我们说到路由的正则编译，正则编译的目的就是和请求的 `url` 来匹配，只有匹配上的路由才是我们真正想要的，此外也会通过正则匹配来获取路由的参数。


# 路由的匹配

路由进行正则编译后，就要与请求 `request` 来进行正则匹配，并且进行一些验证，例如 `UriValidator`、`MethodValidator`、`SchemeValidator`、`HostValidator`。

```php
class RouteCollection implements Countable, IteratorAggregate
{
	public function match(Request $request)
    {
        $routes = $this->get($request->getMethod());

        $route = $this->matchAgainstRoutes($routes, $request);

        if (! is_null($route)) {
            return $route->bind($request);
        }

        $others = $this->checkForAlternateVerbs($request);

        if (count($others) > 0) {
            return $this->getRouteForMethods($request, $others);
        }

        throw new NotFoundHttpException;
    }
    
    protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
	{
   		 return Arr::first($routes, function ($value) use ($request, $includingMethod) {
	    	 return $value->matches($request, $includingMethod);
   		 });
	}
}
    
class Route
{
	public function matches(Request $request, $includingMethod = true)
	{
   		 $this->compileRoute();

    	foreach ($this->getValidators() as $validator) {
        	if (! $includingMethod && $validator instanceof MethodValidator) {
           	 continue;
       	 }

       	 if (! $validator->matches($this, $request)) {
           	 return false;
       	 }
   		 }

   		 return true;
	}

	public static function getValidators()
	{
    	if (isset(static::$validators)) {
        	return static::$validators;
    	}

    	return static::$validators = [
        	new UriValidator, new MethodValidator,
        	new SchemeValidator, new HostValidator,
    	];
	}
}
```

## UriValidator uri 验证

`UriValidator` 验证主要是目的是查看路由正则与请求是否匹配：

```php
class UriValidator implements ValidatorInterface
{
    public function matches(Route $route, Request $request)
    {
        $path = $request->path() == '/' ? '/' : '/'.$request->path();

        return preg_match($route->getCompiled()->getRegex(), rawurldecode($path));
    }
}
```

值得注意的是，在匹配路径之前，程序使用了 `rawurldecode` 来对请求进行解码。

## MethodValidator 验证

请求方法验证：

```php
class MethodValidator implements ValidatorInterface
{
    public function matches(Route $route, Request $request)
    {
        return in_array($request->getMethod(), $route->methods());
    }
}
```

## SchemeValidator 验证

路由 `scheme` 协议验证：

```php
class SchemeValidator implements ValidatorInterface
{
    public function matches(Route $route, Request $request)
    {
        if ($route->httpOnly()) {
            return ! $request->secure();
        } elseif ($route->secure()) {
            return $request->secure();
        }

        return true;
    }
}

public function httpOnly()
{
    return in_array('http', $this->action, true);
}

public function secure()
{
    return in_array('https', $this->action, true);
}
```

## HostValidator 验证
主域验证：

```php
class HostValidator implements ValidatorInterface
{
    public function matches(Route $route, Request $request)
    {
        if (is_null($route->getCompiled()->getHostRegex())) {
            return true;
        }

        return preg_match($route->getCompiled()->getHostRegex(), $request->getHost());
    }
}
```

也就是说，如果路由中并不设置 `host` 属性，那么这个验证并不进行。

# 路由的参数绑定

一旦某个路由符合请求的 `uri` 四项认证，就将会被返回，接下来就要对路由的参数进行绑定与赋值：

```php
class RouteCollection implements Countable, IteratorAggregate
{
    public function bind(Request $request)
    {
        $this->compileRoute();

        $this->parameters = (new RouteParameterBinder($this))
                        ->parameters($request);

        return $this;
    }
}
```
`bind` 函数负责路由参数与请求 `url` 的绑定工作:

```php
class RouteParameterBinder
{
	public function parameters($request)
    {
        $parameters = $this->bindPathParameters($request);

        if (! is_null($this->route->compiled->getHostRegex())) {
            $parameters = $this->bindHostParameters(
                $request, $parameters
            );
        }

        return $this->replaceDefaults($parameters);
    }
}
```
可以看出，路由参数绑定分为主域参数绑定与路径参数绑定，我们先看路径参数绑定：

## 路径参数绑定

```php
class RouteParameterBinder
{
	protected function bindPathParameters($request)
	{
  		  preg_match($this->route->compiled->getRegex(), '/'.$request->decodedPath(), $matches);

  	 	 return $this->matchToKeys(array_slice($matches, 1));
	}
}
```

例如，`{foo}/{baz?}.{ext?}` 进行正则编译后结果：

```php
#^/(?P<foo>[^/]++)(?:/(?P<baz>[^/\.]++)(?:\.(?P<ext>[^/]++))?)?$#s
```

其与 `request` 匹配后的结果为： 

```php
$matches = array (
    0   = "/foo/baz.ext",
    1   = "foo",
    foo = "foo",
    2   = "baz",
    baz = "baz",
    3   = "ext",
    ext = "ext",
)
```

`array_slice($matches, 1)` 取出了 `$matches` 数组 1 之后的结果，然后调用了 `matchToKeys` 函数，

```php
protected function matchToKeys(array $matches)
{
	 if (empty($parameterNames = $this->route->parameterNames())) {
  		  return [];
	 }

	$parameters = array_intersect_key($matches, array_flip($parameterNames));

	return array_filter($parameters, function ($value) {
   		 return is_string($value) && strlen($value) > 0;
	 });
}
```

该函数中利用正则获取了路由的所有参数：

```php
class Route
{
	public function parameterNames()
    {
        if (isset($this->parameterNames)) {
            return $this->parameterNames;
        }

        return $this->parameterNames = $this->compileParameterNames();
    }

    
    protected function compileParameterNames()
    {
        preg_match_all('/\{(.*?)\}/', $this->domain().$this->uri, $matches);

        return array_map(function ($m) {
            return trim($m, '?');
        }, $matches[1]);
    }
}
```
可以看出，获取路由参数的正则表达式采用了勉强模式，意图提取出所有的路由参数。否则，对于路由 `{foo}/{baz?}.{ext?}`,贪婪型正则表达式 `/\{(.*)\}/` 将会匹配整个字符串，而不是各个参数分组。

提取出的参数结果为： 

```php
$matches = array (
	0 = array (
		0 = "{foo}".
		1 = "{baz?}",
		2 = "{ext?}",
	)
	1 = array (
		0 = "foo".
		1 = "baz?",
		2 = "ext?",
	)
)
```
得出的结果将会去除 `$matches[1]`，并且将会删除结果中最后的 `?`。

之后，在 `matchToKeys` 函数中，

```php
$parameters = array_intersect_key($matches, array_flip($parameterNames));
```
获取了匹配结果与路由所有参数的交集：

 ```php
 $parameters = array (
    foo = "foo",
    baz = "baz",
    ext = "ext",
 )
 ```
 
 
## 主域参数绑定

```php
protected function bindHostParameters($request, $parameters)
{
    preg_match($this->route->compiled->getHostRegex(), $request->getHost(), $matches);

    return array_merge($this->matchToKeys(array_slice($matches, 1)), $parameters);
}
```

步骤与路由参数绑定一致。

## 替换默认值

进行参数绑定后，有一些可选参数并没有在 `request` 中匹配到，这时候就要用可选参数的默认值添加到变量 `parameters` 中：

```php
protected function replaceDefaults(array $parameters)
{
    foreach ($parameters as $key => $value) {
        $parameters[$key] = isset($value) ? $value : Arr::get($this->route->defaults, $key);
    }

    foreach ($this->route->defaults as $key => $value) {
        if (! isset($parameters[$key])) {
            $parameters[$key] = $value;
        }
    }

    return $parameters;
}
```

# 匹配异常处理

如果 `url` 匹配失败，没有找到任何路由与请求相互匹配，就会切换 `method` 方法，以求任意路由来匹配：

```php
protected function checkForAlternateVerbs($request)
{
    $methods = array_diff(Router::$verbs, [$request->getMethod()]);

    $others = [];

    foreach ($methods as $method) {
        if (! is_null($this->matchAgainstRoutes($this->get($method), $request, false))) {
            $others[] = $method;
        }
    }

    return $others;
}
```

如果使用其他方法匹配成功，就要判断当前方法是否是 `options`，如果是则直接返回，否则报出异常：	

```php
protected function getRouteForMethods($request, array $methods)
{
    if ($request->method() == 'OPTIONS') {
        return (new Route('OPTIONS', $request->path(), function () use ($methods) {
            return new Response('', 200, ['Allow' => implode(',', $methods)]);
        }))->bind($request);
    }

    $this->methodNotAllowed($methods);
}

protected function methodNotAllowed(array $others)
{
    throw new MethodNotAllowedHttpException($others);
}
```