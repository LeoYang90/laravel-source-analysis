title: Laravel Http——路由的正则编译
tags: []
categories: []
date: 2017-07-24 17:05:00
---

---
# 前言
利用 `pipeline` 进行中间件的层层处理后，接下来 `laravel` 就会利用请求的 `url` 来寻找与其对应的路由，`laravel` 采用对路由注册的 `uri` 进行正则编译，然后利用 `request` 的 `url` 进行正则匹配来寻找正确的路由。

# 前期准备

在上一篇文章中，我们了解了 `Pipeline` 的原理，我们知道它调用了 `dispatchToRouter()` 这个函数：

```php
protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}

protected function dispatchToRouter()
{
    return function ($request) {
        $this->app->instance('request', $request);

        return $this->router->dispatch($request);
    };
}
```
这个函数实际上利用的是 `Router` 的 `dispatch`,这个函数的任务是进行路由匹配，并且调用路由绑定的控制器或者闭包函数：

```php
class Router implements RegistrarContract, BindingRegistrar
{
    public function dispatch(Request $request)
    {
        $this->currentRequest = $request;

        return $this->dispatchToRoute($request);
    }
    
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
}
```

我们这篇文章就是讲解第一句: `findRoute()` 路由匹配:

```php
protected function findRoute($request)
{
    $this->current = $route = $this->routes->match($request);

    $this->container->instance(Route::class, $route);

    return $route;
}
```
寻找路由的任务由 `RouteCollection` 负责，这个函数负责匹配路由，并且把 `request` 的 `url` 参数绑定到路由中：

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
```

# 路由正则匹配
如何去寻找请求 `request` 想要调用的路由呢？ `laravel` 首先对路由进行正则编译，得到路由的正则匹配串，然后利用请求的 `url` 尝试去匹配，如果匹配成功，那么就会选定该路由：

```php
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
    
    protected function compileRoute()
    {
        if (! $this->compiled) {
            $this->compiled = (new RouteCompiler($this))->compile();
        }

       return $this->compiled;
    }
}
```

可以看出，路由的正则编译由 `RouteCompiler` 类专门负责：

```php
class RouteCompiler
{
     public function __construct($route)
    {
        $this->route = $route;
    }
    
    public function compile()
    {
        $optionals = $this->getOptionalParameters();

        $uri = preg_replace('/\{(\w+?)\?\}/', '{$1}', $this->route->uri());

        return (
            new SymfonyRoute($uri, $optionals, $this->route->wheres, [], $this->route->domain() ?: '')
        )->compile();
    }
}
```

可以看出， `laravel` 真正的正则编译是重用 `symfony` 框架的，但是在利用 `symfony` 进行正则编译之前，`laravel` 先对路由的 `uri` 进行了一些处理，以适应 `symfony` 的要求。

## 路由可选参数转换

对于 `laravel` 来说，可以选择某个路由 `url` 的参数是可选的，通常来说，这种可选参数都有默认值。 `laravel` 利用 `?` 来表示可选参数：

```php
$router->get('{foo?}/{baz?}', function ($name = 'taylor', $age = 25) {
     return $name.$age;
});
```
但是对于 `symfony` 来说， `?` 没有任何特殊意义， `symfony` 利用 `SymfonyRoute` 类进行路由初始化，并把第二个参数作为可选参数，因此 `laravel` 需要把可选参数提取出来，然后赋给 `SymfonyRoute` 构造函数。

可选参数的提取由 `getOptionalParameters` 负责：

```php
 protected function getOptionalParameters()
{
    preg_match_all('/\{(\w+?)\?\}/', $this->route->uri(), $matches);

    return isset($matches[1]) ? array_fill_keys($matches[1], null) : [];
}
```

>  `preg_match_all` 函数用于进行正则表达式全局匹配，成功返回整个模式匹配的次数（可能为零），如果出错返回 FALSE。 

> 默认排序方式为 `PREG_PATTERN_ORDER`,结果排序为 `$matches[0]` 保存完整模式的所有匹配, `$matches[1]` 保存第一个子组的所有匹配，以此类推。

> 若排序方式为 `PREG_SET_ORDER`,结果排序为 `$matches[0]` 包含第一次匹配得到的所有匹配(包含子组)， `$matches[1]` 是包含第二次匹配到的所有匹配(包含子组)的数组，以此类推。


以 `{foo?}/{baz?}` 为例，得到的 `matches[0]`:

```php
matches[0] = array (
    0 = '{foo?}',
    1 = '{baz?}',
)
```
得到的结果 `matches` 中 `matches[1]` 是被匹配上的字符串，以 `{foo?}/{baz?}` 为例，得到的 `matches[1]`:

```php
matches[1] = array (
    0 = 'foo',
    1 = 'baz',
)
```

`array_fill_keys` 函数负责使用指定的键和值填充数组，例如上例中就可以得到：

```php
optionals = array (
    foo = null,
    baz = null,
)
```
得到可选参数的数组 `optionals` 后，就要将路由的 `uri` 中 `?` 替换掉，这也就是 `preg_replace` 的作用，以 `{foo?}/{baz?}` 为例，最后得到的替换结果为 `{foo}/{baz}`。


## Symfony 路由初始化

在 `symfony` 的路由初始化中，由很多参数：

- path 是路由的 `uri`
- defaults 是路由可选参数
- requirements 是路由的参数正则约束
- options 路由的选项参数，例如路由正则编译类等
- host 是路由的主域
- schenes 是 web 的协议，例如 http, https
- methods 是调用的方法，例如 `get`、`post`
- condition 

```php
namespace Symfony\Component\Routing;

class Route implements \Serializable
{
    public function __construct($path, array $defaults = array(), array $requirements = array(), array $options = array(), $host = '', $schemes = array(), $methods = array(), $condition = '')
    {
        $this->setPath($path);
        $this->setDefaults($defaults);
        $this->setRequirements($requirements);
        $this->setOptions($options);
        $this->setHost($host);
        $this->setSchemes($schemes);
        $this->setMethods($methods);
        $this->setCondition($condition);
    }
    
    public function setOptions(array $options)
    {
        $this->options = array(
            'compiler_class' => 'Symfony\\Component\\Routing\\RouteCompiler',
        );

        return $this->addOptions($options);
    }
}
```

可以看出， `laravel` 初始化路由的时候，分别初始化了 `path`、`defaults`、`requirements`、`host`，其余都是默认值。其中 `host` 是路由的 `domain` 去除 `http`、`https` 之后的主域。

```php
public function domain()
{
    return isset($this->action['domain'])
            ? str_replace(['http://', 'https://'], '', $this->action['domain']) : null;
}
```
## 路由的正则编译
路由的编译由 `symfony` 的 `route` 类完成：

```php
public function compile()
{
    if (null !== $this->compiled) {
        return $this->compiled;
    }

    $class = $this->getOption('compiler_class');

    return $this->compiled = $class::compile($this);
}
```
`compiler_class` 是初始化的时候提供的类 `Symfony\\Component\\Routing\\RouteCompiler`.

下面是就是路由编译的主要功能实现：

### compile 函数

```php
namespace Symfony\Component\Routing;

class RouteCompiler implements RouteCompilerInterface
{
    public static function compile(Route $route)
    {
        $hostVariables = array();
        $variables = array();
        $hostRegex = null;
        $hostTokens = array();

        if ('' !== $host = $route->getHost()) {
            $result = self::compilePattern($route, $host, true);

            $hostVariables = $result['variables'];
            $variables = $hostVariables;

            $hostTokens = $result['tokens'];
            $hostRegex = $result['regex'];
        }

        $path = $route->getPath();

        $result = self::compilePattern($route, $path, false);

        $staticPrefix = $result['staticPrefix'];

        $pathVariables = $result['variables'];

        foreach ($pathVariables as $pathParam) {
            if ('_fragment' === $pathParam) {
                throw new \InvalidArgumentException(sprintf('Route pattern "%s" cannot contain "_fragment" as a path parameter.', $route->getPath()));
            }
        }

        $variables = array_merge($variables, $pathVariables);

        $tokens = $result['tokens'];
        $regex = $result['regex'];

        return new CompiledRoute(
            $staticPrefix,
            $regex,
            $tokens,
            $pathVariables,
            $hostRegex,
            $hostTokens,
            $hostVariables,
            array_unique($variables)
        );
    }
}
```

可以看出，路由的正则编译由两个部分构成：主域的正则编译与 `uri` 的正则编译。这两个部分的编译功能由函数 `compilePattern` 负责，这个函数会有返回三种数据结果，以 `/foo/{bar}` 为例： 

- `variables` 代表正则匹配的路由参数,如 `bar`
- `tokens` 代表正则匹配的普通路由字符串,如 `foo`
- `regex` 代表路由匹配的正则表达式结果
- 有时候也会有 `$staticPrefix`,这个是路由 `url` 前没有路由参数的字符串前缀，如 `/foo/`.

### compilePattern 函数

由于 `symfony` 原始的正则编译稍微复杂，本文剔除了一些处理 `utf8` 和异常处理的代码，特意挑选计算正则表达式的主干代码，如下：

```php
	private static function compilePattern(Route $route, $pattern, $isHost)
    {
        $tokens = array();
        $variables = array();
        $matches = array();
        $pos = 0;
        $defaultSeparator = $isHost ? '.' : '/';
       
        preg_match_all('#\{\w+\}#', $pattern, $matches, PREG_OFFSET_CAPTURE | PREG_SET_ORDER);
        foreach ($matches as $match) {
            $varName = substr($match[0][0], 1, -1);
            $precedingText = substr($pattern, $pos, $match[0][1] - $pos);
            $pos = $match[0][1] + strlen($match[0][0]);

            if (!strlen($precedingText)) {
                $precedingChar = '';
            } else {
                $precedingChar = substr($precedingText, -1);
            }
            $isSeparator = '' !== $precedingChar && false !== strpos(static::SEPARATORS, $precedingChar);

            if ($isSeparator && $precedingText !== $precedingChar) {
                $tokens[] = array('text', substr($precedingText, 0, -strlen($precedingChar)));
            } elseif (!$isSeparator && strlen($precedingText) > 0) {
                $tokens[] = array('text', $precedingText);
            }

            $regexp = $route->getRequirement($varName);
            if (null === $regexp) {
                $followingPattern = (string) substr($pattern, $pos);
                
                $nextSeparator = self::findNextSeparator($followingPattern, $useUtf8);
                $regexp = sprintf(
                    '[^%s%s]+',
                    preg_quote($defaultSeparator, self::REGEX_DELIMITER),
                    $defaultSeparator !== $nextSeparator && '' !== $nextSeparator ? preg_quote($nextSeparator, self::REGEX_DELIMITER) : ''
                );
                if (('' !== $nextSeparator && !preg_match('#^\{\w+\}#', $followingPattern)) || '' === $followingPattern) {
                    $regexp .= '+';
                }
            }

            $tokens[] = array('variable', $isSeparator ? $precedingChar : '', $regexp, $varName);
            $variables[] = $varName;
        }

        if ($pos < strlen($pattern)) {
            $tokens[] = array('text', substr($pattern, $pos));
        }

        // find the first optional token
        $firstOptional = PHP_INT_MAX;
        if (!$isHost) {
            for ($i = count($tokens) - 1; $i >= 0; --$i) {
                $token = $tokens[$i];
                if ('variable' === $token[0] && $route->hasDefault($token[3])) {
                    $firstOptional = $i;
                } else {
                    break;
                }
            }
        }

        // compute the matching regexp
        $regexp = '';
        for ($i = 0, $nbToken = count($tokens); $i < $nbToken; ++$i) {
            $regexp .= self::computeRegexp($tokens, $i, $firstOptional);
        }
        $regexp = self::REGEX_DELIMITER.'^'.$regexp.'$'.self::REGEX_DELIMITER.'s'.($isHost ? 'i' : '');

        return array(
            'staticPrefix' => 'text' === $tokens[0][0] ? $tokens[0][1] : '',
            'regex' => $regexp,
            'tokens' => array_reverse($tokens),
            'variables' => $variables,
        );
    }
```

下面本文将以 `prefix/{foo}/{baz}.{ext}/tail` 为例，来详细讲一下路由 `uri` 的正则编译过程。

### `preg_match_all` 全匹配

由于 `preg_match_all` 使用了 `PREG_SET_ORDER`,因此结果数组 `matches` 中每一个元素都是一次匹配的结果，本例中：

```php
$matches = array (
	0 = array (
		0 = array (
			0 = "{foo}",
			1 = 8
		)
	)
    1 = array (
		0 = array (
			0 = "{baz}",
			1 = 14
		)
	)
	2 = array (
		0 = array (
			0 = "{ext}",
			1 = 20
		)
	)
)
```
接下来，程序会用循环来分别处理各个匹配的结果。

### 变量
每个匹配结果都会先计算变量: `varName`、`precedingText`、`precedingChar`、`isSeparator`

- `varName` 匹配结果会将路由参数提取出来，本例中：`foo`、`baz`、`ext`
- `precedingText` 是两个路由参数之间的字符串，本例中：`prefix/`、`/`、`.`
- `precedingChar` 是每个路由参数之前的字符，也就是 `precedingText` 的最后一个字符,本例中：`/`、`/`、`.`
- `isSeparator` 判断 `precedingChar` 是否是 `url` 的间隔符，本例中：`true`、`true`、`true`

### tokens-text

将 `precedingText` 记录进 `tokens` 数组，key 为 `text`。
第一次循环，tokens：

```php
tokens = array (
	0 = text,
	1 = prefix,
)
```

第二次循环与第三次循环由于 `precedingText` == `precedingChar`，所以并不会记录。

### 构建 regexp

若在路由定义的过程中利用 `where` 属性或者 `pattern` 为路由的参数设置正则约束，那么此时就会将约束规则赋给 `regexp`，否则就会启用构建 `regexp` 的过程：

```php
$followingPattern = (string) substr($pattern, $pos);
$nextSeparator = self::findNextSeparator($followingPattern, $useUtf8);

$regexp = sprintf(
            '[^%s%s]+',
            preg_quote($defaultSeparator, self::REGEX_DELIMITER),
            $defaultSeparator !== $nextSeparator && '' !== $nextSeparator ? preg_quote($nextSeparator, self::REGEX_DELIMITER) : ''
);

if (('' !== $nextSeparator && !preg_match('#^\{\w+\}#', $followingPattern)) || '' === $followingPattern) {    
    $regexp .= '+';
}
```

构建  `regexp` 有两个部分，

- 寻找 `nextSeparator` ：

```php
private static function findNextSeparator($pattern, $useUtf8)
{
    if ('' == $pattern) {
        return '';
    }
       
    if ('' === $pattern = preg_replace('#\{\w+\}#', '', $pattern)) {
        return '';
    }

    return false !== strpos(static::SEPARATORS, $pattern[0]) ? $pattern[0] : '';
}
```

这个函数的意义在于为路由的 `uri` 的路由参数寻找非默认间隔符，例如，路由可以这样设置 `uri` ：

```php
/{baz}.{ext}/
```
默认的间隔符就是 `/`，如果不设置非默认间隔符的时候，那么 `regexp = [^/]`，`mobile.html` 这样的请求就会被 `{baz}` 这个参数全部匹配到，`{ext}` 就没有任何参数来对应。设置了非默认间隔符后 `regexp = [^/.]`, `baz` 就会匹配 `mobile`，`ext` 就会匹配 `html`。

- 侵占型正则表达式

```php
if (('' !== $nextSeparator && !preg_match('#^\{\w+\}#', $followingPattern)) || '' === $followingPattern) {
    $regexp .= '+';
}
```

为了减少贪婪型正则表达式的回溯导致的性能浪费，当后续字符串已经结束或者不存在 `/{x}{y}` 这样情况的时候，程序将贪婪型正则表达式改为侵占型正则表达式。有关正则表达式的模式请查看：[正则表达式之 贪婪与非贪婪模式详解（概述）](http://blog.csdn.net/u014762221/article/details/68953155)

### tokens-variable

获取路由参数和正则表达式之后，就要更新 `tokens`,分别将 `isSeparator`, `regexp`, `varName` 更新到结果数组中。

以 `prefix/{foo}/{baz}.{ext}/tail` 为例，`$tokens` 在各个循环时值为：

```php
$tokens = array (
	0 = array (
		0 = ‘text’,
		1 = '/prefix'
	)
	1 = array (
		0 = ‘variable’,
		1 = '/',
		0 = ‘[^/]++’,
		1 = 'foo'
	)//第一次循环结束
	2 = array (
		0 = ‘variable’,
		1 = '/',
		0 = ‘[^/\.]++’,
		1 = 'baz'
	)//第二次循环结束
	3 = array (
		0 = ‘variable’,
		1 = '.',
		0 = ‘[^/]++’,
		1 = 'ext'
	)//循环结束
	4 = array (
		0 = ‘text’,
		1 = '/tail'
	)//	循环外
)

```

### 默认路由参数

接下来就要计算首个默认路由参数在整个路由 `url` 的位置，以便在生成正则表达式中使用：

```php
$firstOptional = PHP_INT_MAX;
if (!$isHost) {
    for ($i = count($tokens) - 1; $i >= 0; --$i) {
        $token = $tokens[$i];
        if ('variable' === $token[0] && $route->hasDefault($token[3])) {
            $firstOptional = $i;
        } else {
            break;
        }
    }
}
```

### 计算正则表达式
所有的 `tokens` 数组都构建完毕，接下来就需要利用这个数组来构建正则表达式了。

```php
$regexp = '';
for ($i = 0, $nbToken = count($tokens); $i < $nbToken; ++$i) {
    $regexp .= self::computeRegexp($tokens, $i, $firstOptional);
}
$regexp = self::REGEX_DELIMITER.'^'.$regexp.'$'.self::REGEX_DELIMITER.'s'.($isHost ? 'i' : '');
```

```php
    private static function computeRegexp(array $tokens, $index, $firstOptional)
    {
        $token = $tokens[$index];
        if ('text' === $token[0]) {
            // Text tokens
            return preg_quote($token[1], self::REGEX_DELIMITER);
        } else {
            // Variable tokens
            if (0 === $index && 0 === $firstOptional) {
                // When the only token is an optional variable token, the separator is required
                return sprintf('%s(?P<%s>%s)?', preg_quote($token[1], self::REGEX_DELIMITER), $token[3], $token[2]);
            } else {
                $regexp = sprintf('%s(?P<%s>%s)', preg_quote($token[1], self::REGEX_DELIMITER), $token[3], $token[2]);
                if ($index >= $firstOptional) {
                    $regexp = "(?:$regexp";
                    $nbTokens = count($tokens);
                    if ($nbTokens - 1 == $index) {
                        // Close the optional subpatterns
                        $regexp .= str_repeat(')?', $nbTokens - $firstOptional - (0 === $firstOptional ? 1 : 0));
                    }
                }

                return $regexp;
            }
        }
    }
```

`computeRegexp` 函数的大致流程为：

- 若 `tokens` 当前元素是 `text` ，不是路由参数的时候，直接赋值原字符串即可
- 若 `url` 中路由参数都是可选参数，且没有任何 `text`，那么第一个可选参数使用捕获分组
- 若当前路由参数是可选参数的时候，需要在正则表达式中不断叠加非捕获分组 `(?`，再最后设置为可选分组 `)?`，例如 `(?:/(?P<baz>[^/]++)(?:/(?P<ext>[^/]++))?)?`
- 若当前路由参数不是可选参数的时候，正则表达式就是固定模式，例如： `/(?P<foo>[^/]++)`

利用 `computeRegexp` 函数拼接正则表达式后，还要在最两侧分隔符、开始符 `^`,结束符 `$`、单行修正符 `s`，如果是主域的正则表达式，还要添加不区分大小写的修正符 `i`。

以 `prefix/{foo}/{baz}.{ext}/tail` 为例,每次生成的正则表达式如下：

```php
/prefix
/prefix/(?P<foo>[^/]++)
/prefix/(?P<foo>[^/]++)/(?P<baz>[^/\.]++)
/prefix/(?P<foo>[^/]++)/(?P<baz>[^/\.]++)\.(?P<ext>[^/]++)
/prefix/(?P<foo>[^/]++)/(?P<baz>[^/\.]++)\.(?P<ext>[^/]++)/tail
#^/prefix/(?P<foo>[^/]++)/(?P<baz>[^/\.]++)\.(?P<ext>[^/]++)/tail$#s
```
以 `{foo?}/{baz?}.{ext?}` 为例,每次生成的正则表达式如下：

```php
/(?P<foo>[^/]++)?
/(?P<foo>[^/]++)?(?:/(?P<baz>[^/\.]++)
/(?P<foo>[^/]++)?(?:/(?P<baz>[^/\.]++)(?:\.(?P<ext>[^/]++))?)?
#^/(?P<foo>[^/]++)?(?:/(?P<baz>[^/\.]++)(?:\.(?P<ext>[^/]++))?)?$#s

```