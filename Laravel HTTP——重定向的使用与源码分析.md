title: Laravel HTTP——重定向的使用与源码分析
date: 2017-08-07 11:12:39
---

---
# 前言
`laravel` 为我们提供便携的重定向功能，可以由门面 `Redirect`，或者全局函数 `redirect()` 来启用，本篇文章将会介绍重定向功能的具体细节及源码分析。

# URI 重定向

重定向功能是由类 `UrlGenerator` 所实现，这个类需要 `request` 来进行初始化：

```php
 $url = new UrlGenerator(
    $routes = new RouteCollection,
    $request = Request::create('http://www.foo.com/')
);

```

## 重定向到 uri

- 当我们想要重定向到某个地址时，可以使用 `to` 函数：

```php
$this->assertEquals('http://www.foo.com/foo/bar', $url->to('foo/bar'));
```

- 当我们想要添加额外的路径，可以将数组赋给第二个参数：

```php
$this->assertEquals('https://www.foo.com/foo/bar/baz/boom', $url->to('foo/bar', ['baz', 'boom'], true));
$this->assertEquals('https://www.foo.com/foo/bar/baz?foo=bar', $url->to('foo/bar?foo=bar', ['baz'], true));

```
## 强制 https

如果我们想要重定向到 `https` ，我们可以设置第三个参数为 `true` :

```php
$this->assertEquals('https://www.foo.com/foo/bar', $url->to('foo/bar', [], true));
```

或者使用 `forceScheme` 函数：

```php
$url->forceScheme('https');

$this->assertEquals('https://www.foo.com/foo/bar', $url->to('foo/bar');
```

## 强制域名

```php
$url->forceRootUrl('https://www.bar.com');

$this->assertEquals('https://www.bar.com/foo/bar', $url->to('foo/bar');
```

## 路径自定义

```php
$url->formatPathUsing(function ($path) {
    return '/something'.$path;
});

$this->assertEquals('http://www.foo.com/something/foo/bar', $url->to('foo/bar'));
```

# 路由重定向

重定向另一个非常重要的功能是重定向到路由所在的地址中去：

```php
$route = new Route(['GET'], '/named-route', ['as' => 'plain']);
$routes->add($route);

$this->assertEquals('http:/www.bar.com/named-route', $url->route('plain'));
```

## 非域名路径

`laravel` 路由重定向可以选择重定向后的地址是否仍然带有域名，这个特性由第三个参数决定：

```php
$route = new Route(['GET'], '/named-route', ['as' => 'plain']);
$routes->add($route);

$this->assertEquals('/named-route', $url->route('plain', [], false));
```

## 重定向端口号

路由重定向可以允许带有 `request` 自己的端口：

```php
$url = new UrlGenerator(
    $routes = new RouteCollection,
    $request = Request::create('http://www.foo.com:8080/')
);

$route = new Route(['GET'], 'foo/bar/{baz}', ['as' => 'bar', 'domain' => 'sub.{foo}.com']);
$routes->add($route);

$this->assertEquals('http://sub.taylor.com:8080/foo/bar/otwell', $url->route('bar', ['taylor', 'otwell']));
```

## 重定向路径参数绑定

如果路由中含有参数，可以将需要的参数赋给 `route` 第二个参数：

```php
$route = new Route(['GET'], 'foo/bar/{baz}', ['as' => 'foobar']);
$routes->add($route);

$this->assertEquals('http://www.foo.com/foo/bar/taylor', $url->route('foobar', 'taylor'));
```

也可以根据参数的命名来指定参数绑定：

```php
$route = new Route(['GET'], 'foo/bar/{baz}/breeze/{boom}', ['as' => 'bar']);
$routes->add($route);

$this->assertEquals('http://www.foo.com/foo/bar/otwell/breeze/taylor', $url->route('bar', ['boom' => 'taylor', 'baz' => 'otwell']));
```

还可以利用 `defaults` 函数为重定向提供默认的参数来绑定：

```php
$url->defaults(['locale' => 'en']);
$route = new Route(['GET'], 'foo', ['as' => 'defaults', 'domain' => '{locale}.example.com', function () {
}]);
$routes->add($route);

$this->assertEquals('http://en.example.com/foo', $url->route('defaults'));
```

## 重定向路由 querystring 添加

当在 `route` 函数中赋给的参数多于路径参数的时候，多余的参数会被添加到 `querystring` 中：

```php
$route = new Route(['GET'], 'foo/bar/{baz}/breeze/{boom}', ['as' => 'bar']);
$routes->add($route);

$this->assertEquals('http://www.foo.com/foo/bar/taylor/breeze/otwell?fly=wall', $url->route('bar', ['taylor', 'otwell', 'fly' => 'wall']));
```

## fragment 重定向

```php
$route = new Route(['GET'], 'foo/bar#derp', ['as' => 'fragment']);
$routes->add($route);

$this->assertEquals('/foo/bar?baz=%C3%A5%CE%B1%D1%84#derp', $url->route('fragment', ['baz' => 'åαф'], false));
```

## 路由 action 重定向

我们不仅可以通过路由的别名来重定向，还可以利用路由的控制器方法来重定向：

```php
$route = new Route(['GET'], 'foo/bam', ['controller' => 'foo@bar']);
$routes->add($route);
   
$this->assertEquals('http://www.foo.com/foo/bam', $url->action('foo@bar'));        
```
可以设定重定向控制器的默认命名空间：

```php
$url->setRootControllerNamespace('namespace');

$route = new Route(['GET'], 'foo/bar', ['controller' => 'namespace\foo@bar']);
        $routes->add($route);

$route = new Route(['GET'], 'something/else', ['controller' => 'something\foo@bar']);
$routes->add($route);

$this->assertEquals('http://www.foo.com/foo/bar', $url->action('foo@bar'));
$this->assertEquals('http://www.foo.com/something/else', $url->action('\something\foo@bar'));
```

## UrlRoutable 参数绑定

可以为重定向传入 `UrlRoutable` 类型的参数，重定向会通过类方法 `getRouteKey` 来获取对象的某个属性，进而绑定到路由的参数中去。

```php
public function testRoutableInterfaceRoutingWithSingleParameter()
{
    $url = new UrlGenerator(
        $routes = new RouteCollection,
        $request = Request::create('http://www.foo.com/')
    );

    $route = new Route(['GET'], 'foo/{bar}', ['as' => 'routable']);
    $routes->add($route);

    $model = new RoutableInterfaceStub;
    $model->key = 'routable';

    $this->assertEquals('/foo/routable', $url->route('routable', $model, false));
}

class RoutableInterfaceStub implements UrlRoutable
{
    public $key;

    public function getRouteKey()
    {
        return $this->{$this->getRouteKeyName()};
    }

    public function getRouteKeyName()
    {
        return 'key';
    }
}

```

# URI 重定向源码分析

在说重定向的源码之前，我们先了解一下一般的 `uri` 基本组成：

`scheme://domain:port/path?queryString`

也就是说，一般 `uri` 由五部分构成。重定向实际上就是按照各种传入的参数以及属性的设置来重新生成上面的五部分：

```php
public function to($path, $extra = [], $secure = null)
{
    if ($this->isValidUrl($path)) {
        return $path;
    }

    $tail = implode('/', array_map(
        'rawurlencode', (array) $this->formatParameters($extra))
    );

    $root = $this->formatRoot($this->formatScheme($secure));

    list($path, $query) = $this->extractQueryString($path);

    return $this->format(
        $root, '/'.trim($path.'/'.$tail, '/')
    ).$query;
}
```

## 重定向 scheme

重定向的 `scheme` 由函数 `formatScheme` 生成：

```php
public function formatScheme($secure)
{
    if (! is_null($secure)) {
        return $secure ? 'https://' : 'http://';
    }

    if (is_null($this->cachedSchema)) {
        $this->cachedSchema = $this->forceScheme ?: $this->request->getScheme().'://';
    }

    return $this->cachedSchema;
}

public function forceScheme($schema)
{
    $this->cachedSchema = null;

    $this->forceScheme = $schema.'://';
}
```

可以看出来， `scheme` 的生成存在优先级：

- 由 `to` 传入的 `secure` 参数
- 由 `forceScheme` 设置的 `schema` 参数
- `request` 自带的 `scheme`

## 重定向 domain

重定向的 `domain` 由函数 `formatRoot` 生成：

```php
public function formatRoot($scheme, $root = null)
{
    if (is_null($root)) {
        if (is_null($this->cachedRoot)) {
            $this->cachedRoot = $this->forcedRoot ?: $this->request->root();
        }

        $root = $this->cachedRoot;
    }

    $start = Str::startsWith($root, 'http://') ? 'http://' : 'https://';

    return preg_replace('~'.$start.'~', $scheme, $root, 1);
}

public function forceRootUrl($root)
{
    $this->forcedRoot = rtrim($root, '/');

    $this->cachedRoot = null;
}
```

与 `scheme` 类似，`root` 的生成也存在优先级：

- 由 `to` 传入的 `root` 参数
- 由 `forceRootUrl` 设置的 `root` 参数
- `request` 自带的 `root`

## 重定向 path

重定向的 `path` 由三部分构成，一部分是 `request` 自带的 `path`，一部分是函数 `to` 原有的 `path` ，另一部分是函数 `to` 传入的参数：

```php
public function formatParameters($parameters)
{
    $parameters = array_wrap($parameters);

    foreach ($parameters as $key => $parameter) {
        if ($parameter instanceof UrlRoutable) {
            $parameters[$key] = $parameter->getRouteKey();
        }
    }

    return $parameters;
}

protected function extractQueryString($path)
{
    if (($queryPosition = strpos($path, '?')) !== false) {
        return [
            substr($path, 0, $queryPosition),
            substr($path, $queryPosition),
        ];
    }

    return [$path, ''];
}
```

# 路由重定向源码分析

相对于 `uri` 的重定向来说，路由重定向的 `scheme`、`root` 、`path`、`queryString` 都要以路由自身的属性为第一优先级，此外还要利用额外参数来绑定路由的 `uri` 参数：

```php
 public function route($name, $parameters = [], $absolute = true)
{
    if (! is_null($route = $this->routes->getByName($name))) {
        return $this->toRoute($route, $parameters, $absolute);
    }

    throw new InvalidArgumentException("Route [{$name}] not defined.");
}

public function to($route, $parameters = [], $absolute = false)
{
    $domain = $this->getRouteDomain($route, $parameters);

    $uri = $this->addQueryString($this->url->format(
        $root = $this->replaceRootParameters($route, $domain, $parameters),
        $this->replaceRouteParameters($route->uri(), $parameters)
    ), $parameters);

    if (preg_match('/\{.*?\}/', $uri)) {
        throw UrlGenerationException::forMissingParameters($route);
    }

    $uri = strtr(rawurlencode($uri), $this->dontEncode);

    if (! $absolute) {
        return '/'.ltrim(str_replace($root, '', $uri), '/');
    }

    return $uri;
}
```

## 路由重定向 scheme

路由的重定向 `scheme` 需要先判断路由的 `scheme` 属性：

```php
protected function getRouteScheme($route)
{
    if ($route->httpOnly()) {
        return 'http://';
    } elseif ($route->httpsOnly()) {
        return 'https://';
    } else {
        return $this->url->formatScheme(null);
    }
}
```

## 路由重定向 domain

```php

public function to($route, $parameters = [], $absolute = false)
{
	$domain = $this->getRouteDomain($route, $parameters);

    $uri = $this->addQueryString($this->url->format(
        $root = $this->replaceRootParameters($route, $domain, $parameters),
        $this->replaceRouteParameters($route->uri(), $parameters)
    ), $parameters);
	...
}

protected function getRouteDomain($route, &$parameters)
{
    return $route->domain() ? $this->formatDomain($route, $parameters) : null;
}

protected function formatDomain($route, &$parameters)
{
    return $this->addPortToDomain(
        $this->getRouteScheme($route).$route->domain()
    );
}

protected function addPortToDomain($domain)
{
    $secure = $this->request->isSecure();

    $port = (int) $this->request->getPort();

    return ($secure && $port === 443) || (! $secure && $port === 80)
            ? $domain : $domain.':'.$port;
}

protected function replaceRootParameters($route, $domain, &$parameters)
{
    $scheme = $this->getRouteScheme($route);

    return $this->replaceRouteParameters(
        $this->url->formatRoot($scheme, $domain), $parameters
    );
}
```

可以看出路由重定向时，域名的生成主要先经过函数 `getRouteDomain`, 判断路由是否有 `domain` 属性,如果有域名属性，则将会作为 `formatRoot` 函数的参数传入，否则就会默认启动 1`uri` 重定向的域名生成方法。

## 路由重定向参数绑定

路由重定向可以利用函数 `replaceRootParameters` 在域名当中参数绑定，，也可以在路径当中利用函数 `replaceRouteParameters` 进行参数绑定。参数绑定分为命名参数绑定与匿名参数绑定：

```php
protected function replaceRouteParameters($path, array &$parameters)
{
    $path = $this->replaceNamedParameters($path, $parameters);

    $path = preg_replace_callback('/\{.*?\}/', function ($match) use (&$parameters) {
        return (empty($parameters) && ! Str::endsWith($match[0], '?}'))
                    ? $match[0]
                    : array_shift($parameters);
    }, $path);

    return trim(preg_replace('/\{.*?\?\}/', '', $path), '/');
}
``` 

对于命名参数绑定,程序会分别从变量列表、默认变量列表中获取并替换路由参数对应的数值，若不存在该参数，则直接返回：

```php
protected function replaceNamedParameters($path, &$parameters)
{
    return preg_replace_callback('/\{(.*?)\??\}/', function ($m) use (&$parameters) {
        if (isset($parameters[$m[1]])) {
            return Arr::pull($parameters, $m[1]);
        } elseif (isset($this->defaultParameters[$m[1]])) {
            return $this->defaultParameters[$m[1]];
        } else {
            return $m[0];
        }
    }, $path);
}
```
命名参数绑定结束后，剩下的未被替换的路由参数将会被未命名的变量按顺序来替换。

## 路由重定向 queryString

如果变量列表在绑定路由后仍然有剩余，那么变量将会作为路由的 `queryString`：

```php
protected function addQueryString($uri, array $parameters)
{
    if (! is_null($fragment = parse_url($uri, PHP_URL_FRAGMENT))) {
        $uri = preg_replace('/#.*/', '', $uri);
    }

    $uri .= $this->getRouteQueryString($parameters);

    return is_null($fragment) ? $uri : $uri."#{$fragment}";
}

protected function getRouteQueryString(array $parameters)
{
    if (count($parameters) == 0) {
        return '';
    }

    $query = http_build_query(
        $keyed = $this->getStringParameters($parameters)
    );

        
    if (count($keyed) < count($parameters)) {
        $query .= '&'.implode(
            '&', $this->getNumericParameters($parameters)
        );
    }

    return '?'.trim($query, '&');
}
```

## 路由重定向结束

路由 `uri` 构建完成后，将会继续判断是否存在违背绑定的路由参数，是否显示 `absolute` 的路由地址

```php
public function to($route, $parameters = [], $absolute = false)
{
	...
	if (preg_match('/\{.*?\}/', $uri)) {
        throw UrlGenerationException::forMissingParameters($route);
    }
    
    $uri = strtr(rawurlencode($uri), $this->dontEncode);

    if (! $absolute) {
        return '/'.ltrim(str_replace($root, '', $uri), '/');
    }

    return $uri;
}
```