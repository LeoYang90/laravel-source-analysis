# 前言

对于一个优秀的框架来说，正确的异常处理可以防止暴露自身接口给用户，可以提供快速追溯问题的提示给开发人员。本文会详细的介绍 `laravel` 异常处理的源码。

# PHP 异常处理

本章节参考 [PHP错误异常处理详解](http://blog.csdn.net/hguisu/article/details/7464977)。

异常处理（又称为错误处理）功能提供了处理程序运行时出现的错误或异常情况的方法。
　　
异常处理通常是防止未知错误产生所采取的处理措施。异常处理的好处是你不用再绞尽脑汁去考虑各种错误，这为处理某一类错误提供了一个很有效的方法，使编程效率大大提高。当异常被触发时，通常会发生：

- 当前代码状态被保存
- 代码执行被切换到预定义的异常处理器函数
- 根据情况，处理器也许会从保存的代码状态重新开始执行代码，终止脚本执行，或从代码中另外的位置继续执行脚本
          
PHP 5 提供了一种新的面向对象的错误处理方法。可以使用检测（try）、抛出（throw）和捕获（catch）异常。即使用try检测有没有抛出（throw）异常，若有异常抛出（throw），使用catch捕获异常。

一个 try 至少要有一个与之对应的 catch。定义多个 catch 可以捕获不同的对象。php 会按这些 catch 被定义的顺序执行，直到完成最后一个为止。而在这些 catch 内，又可以抛出新的异常。

## 异常的抛出

当一个异常被抛出时，其后的代码将不会继续执行，PHP 会尝试查找匹配的 `catch` 代码块。如果一个异常没有被捕获，而且又没用使用`set_exception_handler()` 作相应的处理的话，那么 `PHP` 将会产生一个严重的错误，并且输出未能捕获异常 `(Uncaught Exception ... )` 的提示信息。

抛出异常，但不去捕获它：

```php
ini_set('display_errors', 'On');  
error_reporting(E_ALL & ~ E_WARNING);  
$error = 'Always throw this error';  
throw new Exception($error);  
// 继续执行  
echo 'Hello World';  
```
上面的代码会获得类似这样的一个致命错误：

```php
Fatal error: Uncaught exception 'Exception' with message 'Always throw this error' in E:\sngrep\index.php on line 5  
Exception: Always throw this error in E:\sngrep\index.php on line 5  
Call Stack:  
    0.0005     330680   1. {main}() E:\sngrep\index.php:0  
```

## Try, throw 和 catch

要避免上面这个致命错误，可以使用try catch捕获掉。

处理处理程序应当包括：


- Try - 使用异常的函数应该位于 "try" 代码块内。如果没有触发异常，则代码将照常继续执行。但是如果异常被触发，会抛出一个异常。
- Throw - 这里规定如何触发异常。每一个 "throw" 必须对应至少一个 "catch"
- Catch - "catch" 代码块会捕获异常，并创建一个包含异常信息的对象

抛出异常并捕获掉，可以继续执行后面的代码：

```php
try {  
    $error = 'Always throw this error';  
    throw new Exception($error);  
  
    // 从这里开始，tra 代码块内的代码将不会被执行  
    echo 'Never executed';  
  
} catch (Exception $e) {  
    echo 'Caught exception: ',  $e->getMessage(),'<br>';  
}  
  
// 继续执行  
echo 'Hello World';  
```

## 顶层异常处理器 set_exception_handler

在我们实际开发中，异常捕捉仅仅靠 `try {} catch ()` 是远远不够的。`set_exception_handler()` 函数可设置处理所有未捕获异常的用户定义函数。

```php
function myException($exception)  
{  
echo "<b>Exception:</b> " , $exception->getMessage();  
}  
  
set_exception_handler('myException');  
throw new Exception('Uncaught Exception occurred');
```

## 扩展 PHP 内置的异常处理类

用户可以用自定义的异常处理类来扩展 PHP 内置的异常处理类。以下的代码说明了在内置的异常处理类中，哪些属性和方法在子类中是可访问和可继承的。

```php
class Exception  
{  
    protected $message = 'Unknown exception';   // 异常信息  
    protected $code = 0;                        // 用户自定义异常代码  
    protected $file;                            // 发生异常的文件名  
    protected $line;                            // 发生异常的代码行号  
  
    function __construct($message = null, $code = 0);  
  
    final function getMessage();                // 返回异常信息  
    final function getCode();                   // 返回异常代码  
    final function getFile();                   // 返回发生异常的文件名  
    final function getLine();                   // 返回发生异常的代码行号  
    final function getTrace();                  // backtrace() 数组  
    final function getTraceAsString();          // 已格成化成字符串的 getTrace() 信息  
  
    /* 可重载的方法 */  
    function __toString();                       // 可输出的字符串  
}  
```

如果使用自定义的类来扩展内置异常处理类，并且要重新定义构造函数的话，建议同时调用 `parent::__construct()` 来检查所有的变量是否已被赋值。当对象要输出字符串的时候，可以重载 `__toString()` 并自定义输出的样式。 

```php
class MyException extends Exception  
{  
    // 重定义构造器使 message 变为必须被指定的属性  
    public function __construct($message, $code = 0) {  
        // 自定义的代码  
  
        // 确保所有变量都被正确赋值  
        parent::__construct($message, $code);  
    }  
  
    // 自定义字符串输出的样式 */  
    public function __toString() {  
        return __CLASS__ . ": [{$this->code}]: {$this->message}\n";  
    }  
  
    public function customFunction() {  
        echo "A Custom function for this type of exception\n";  
    }  
} 
```

`MyException` 类是作为旧的 exception 类的一个扩展来创建的。这样它就继承了旧类的所有属性和方法，我们可以使用 `exception` 类的方法，比如 `getLine()` 、 `getFile()` 以及 `getMessage()`。

# PHP 错误处理

## PHP 的错误级别

| 值     |   常量    | 说明   |
| :--:   | :--------|:-----|
| 1      | E_ERROR | 致命的运行时错误。这类错误一般是不可恢复的情况，例如内存分配导致的问题。后果是导致脚本终止不再继续运行。|
| 2      | E_WARNING | 运行时警告 (非致命错误)。仅给出提示信息，但是脚本不会终止运行。
| 4      | E_PARSE | 编译时语法解析错误。解析错误仅仅由分析器产生。
| 8      | E_NOTICE | 运行时通知。表示脚本遇到可能会表现为错误的情况，但是在可以正常运行的脚本里面也可能会有类似的通知。
| 16      | E\_CORE\_ERROR | 在PHP初始化启动过程中发生的致命错误。该错误类似 E_ERROR，但是是由PHP引擎核心产生的。
| 32      | E\_CORE\_WARNING | PHP初始化启动过程中发生的警告 (非致命错误) 。类似 E_WARNING，但是是由PHP引擎核心产生的。
| 64      | E\_COMPILE\_ERROR | 致命编译时错误。类似E_ERROR, 但是是由Zend脚本引擎产生的。
| 128      | E\_COMPILE\_WARNING | 编译时警告 (非致命错误)。类似 E_WARNING，但是是由Zend脚本引擎产生的。
| 256      | E\_USER\_ERROR | 用户产生的错误信息。类似 E_ERROR, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
| 512      | E\_USER\_WARNING | 用户产生的警告信息。类似 E_WARNING, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
| 1024      | E\_USER\_NOTICE | 用户产生的通知信息。类似 E_NOTICE, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
| 2048      | E_STRICT | 启用 PHP 对代码的修改建议，以确保代码具有最佳的互操作性和向前兼容性。
| 4096      | E\_RECOVERABLE\_ERROR | 可被捕捉的致命错误。 它表示发生了一个可能非常危险的错误，但是还没有导致PHP引擎处于不稳定的状态。 如果该错误没有被用户自定义句柄捕获 (参见 set_error_handler())，将成为一个 E_ERROR　从而脚本会终止运行。
| 8192      | E_DEPRECATED | 运行时通知。启用后将会对在未来版本中可能无法正常工作的代码给出警告。
| 16384      | E\_USER\_DEPRECATED | 用户产少的警告信息。 类似 E_DEPRECATED, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。
| 30719      | E_ALL | 用户产少的警告信息。 类似 E_DEPRECATED, 但是是由用户自己在代码中使用PHP函数 trigger_error()来产生的。

## 错误的抛出

除了系统在运行 php 代码抛出的意外错误。我们还可以利用 `rigger_error ` 产生一个自定义的用户级别的 `error/warning/notice` 错误信息:

```php
if ($divisor == 0) {
    trigger_error("Cannot divide by zero", E_USER_ERROR);
}
```
## 顶级错误处理器

顶级错误处理器 `set_error_handler` 一般用于捕捉  `E_NOTICE` 、`E_USER_ERROR`、`E_USER_WARNING`、`E_USER_NOTICE` 级别的错误，不能捕捉 `E_ERROR`, `E_PARSE`, `E_CORE_ERROR`, `E_CORE_WARNING`, `E_COMPILE_ERROR` 和`E_COMPILE_WARNING`。

## `register_shutdown_function`

`register_shutdown_function()` 函数可实现当程序执行完成后执行的函数，其功能为可实现程序执行完成的后续操作。程序在运行的时候可能存在执行超时，或强制关闭等情况，但这种情况下默认的提示是非常不友好的，如果使用 `register_shutdown_function()` 函数捕获异常，就能提供更加友好的错误展示方式，同时可以实现一些功能的后续操作，如执行完成后的临时数据清理，包括临时文件等。

可以这样理解调用条件：

- 当页面被用户强制停止时
- 当程序代码运行超时时
- 当ＰＨＰ代码执行完成时，代码执行存在异常和错误、警告

我们前面说过，`set_error_handler` 能够捕捉的错误类型有限，很多致命错误例如解析错误等都无法捕捉，但是这类致命错误发生时，PHP 会调用 `register_shutdown_function` 所注册的函数，如果结合函数 `error_get_last `，就会获取错误发生的信息。

# Laravel 异常处理

`laravel` 的异常处理由类 `\Illuminate\Foundation\Bootstrap\HandleExceptions::class` 完成：

```php
class HandleExceptions
{
	public function bootstrap(Application $app)
    {
        $this->app = $app;

        error_reporting(-1);

        set_error_handler([$this, 'handleError']);

        set_exception_handler([$this, 'handleException']);

        register_shutdown_function([$this, 'handleShutdown']);

        if (! $app->environment('testing')) {
            ini_set('display_errors', 'Off');
        }
    }
}
```

## 异常转化

`laravel` 的异常处理均由函数 `handleException` 负责。 

`PHP7` 实现了一个全局的 `throwable` 接口，原来的 `Exception` 和部分 `Error` 都实现了这个接口， 以接口的方式定义了异常的继承结构。于是，`PHP7` 中更多的 `Error` 变为可捕获的 `Exception` 返回给开发者，如果不进行捕获则为 `Error` ，如果捕获就变为一个可在程序内处理的 `Exception`。这些可被捕获的 `Error` 通常都是不会对程序造成致命伤害的 `Error`，例如函数不存在。

`PHP7` 中，基于 `/Error exception`，派生了5个新的engine exception：`ArithmeticError` / `AssertionError` / `DivisionByZeroError` / `ParseError` / `TypeError`。在 `PHP7` 里，无论是老的 `/Exception` 还是新的 `/Error` ，它们都实现了一个共同的interface: `/Throwable`。

因此，遇到非 `Exception` 类型的异常，首先就要将其转化为 `FatalThrowableError` 类型：

```php
public function handleException($e)
{
    if (! $e instanceof Exception) {
        $e = new FatalThrowableError($e);
    }

    $this->getExceptionHandler()->report($e);

    if ($this->app->runningInConsole()) {
        $this->renderForConsole($e);
    } else {
        $this->renderHttpResponse($e);
    }
}
```

`FatalThrowableError` 是 `Symfony` 继承 `\ErrorException` 的错误异常类：

```php
class FatalThrowableError extends FatalErrorException
{
    public function __construct(\Throwable $e)
    {
        if ($e instanceof \ParseError) {
            $message = 'Parse error: '.$e->getMessage();
            $severity = E_PARSE;
        } elseif ($e instanceof \TypeError) {
            $message = 'Type error: '.$e->getMessage();
            $severity = E_RECOVERABLE_ERROR;
        } else {
            $message = $e->getMessage();
            $severity = E_ERROR;
        }

        \ErrorException::__construct(
            $message,
            $e->getCode(),
            $severity,
            $e->getFile(),
            $e->getLine()
        );

        $this->setTrace($e->getTrace());
    }
}
```

## 异常 Log

当遇到异常情况的时候，`laravel` 首要做的事情就是记录 `log`，这个就是 `report` 函数的作用。

```php
 protected function getExceptionHandler()
{
    return $this->app->make(ExceptionHandler::class);
}
```

`laravel` 在 `Ioc` 容器中默认的异常处理类是 `Illuminate\Foundation\Exceptions\Handler`:

```php
class Handler implements ExceptionHandlerContract
{
    public function report(Exception $e)
    {
        if ($this->shouldntReport($e)) {
            return;
        }

        try {
            $logger = $this->container->make(LoggerInterface::class);
        } catch (Exception $ex) {
            throw $e; // throw the original exception
        }

        $logger->error($e);
    }
    
    protected function shouldntReport(Exception $e)
    {
        $dontReport = array_merge($this->dontReport, [HttpResponseException::class]);

        return ! is_null(collect($dontReport)->first(function ($type) use ($e) {
            return $e instanceof $type;
        }));
    }
}
```

## 异常页面展示

记录 `log` 后，就要将异常转化为页面向开发者展示异常的信息，以便查看问题的来源：

```php
protected function renderHttpResponse(Exception $e)
{
    $this->getExceptionHandler()->render($this->app['request'], $e)->send();
}

class Handler implements ExceptionHandlerContract
{
	public function render($request, Exception $e)
    {
        $e = $this->prepareException($e);

        if ($e instanceof HttpResponseException) {
            return $e->getResponse();
        } elseif ($e instanceof AuthenticationException) {
            return $this->unauthenticated($request, $e);
        } elseif ($e instanceof ValidationException) {
            return $this->convertValidationExceptionToResponse($e, $request);
        }

        return $this->prepareResponse($request, $e);
    }
}
```
对于不同的异常，`laravel` 有不同的处理，大致有 `HttpException`、`HttpResponseException`、`AuthorizationException`、`ModelNotFoundException`、`AuthenticationException`、`ValidationException`。由于特定的不同异常带有自身的不同需求，本文不会特别介绍。本文继续介绍最普通的异常 `HttpException` 的处理：    
    
```php
protected function prepareResponse($request, Exception $e)
{
    if ($this->isHttpException($e)) {
        return $this->toIlluminateResponse($this->renderHttpException($e), $e);
    } else {
        return $this->toIlluminateResponse($this->convertExceptionToResponse($e), $e);
    }
}
    
protected function renderHttpException(HttpException $e)
{
    $status = $e->getStatusCode();

    view()->replaceNamespace('errors', [
        resource_path('views/errors'),
        __DIR__.'/views',
    ]);

    if (view()->exists("errors::{$status}")) {
        return response()->view("errors::{$status}", ['exception' => $e], $status, $e->getHeaders());
    } else {
        return $this->convertExceptionToResponse($e);
    }
}
```

对于 `HttpException` 来说，会根据其错误的状态码，选取不同的错误页面模板，若不存在相关的模板，则会通过 `SymfonyResponse` 来构造异常展示页面：
    
```php
protected function convertExceptionToResponse(Exception $e)
{
    $e = FlattenException::create($e);

    $handler = new SymfonyExceptionHandler(config('app.debug'));

    return SymfonyResponse::create($handler->getHtml($e), $e->getStatusCode(), $e->getHeaders());
}
    
protected function toIlluminateResponse($response, Exception $e)
{
    if ($response instanceof SymfonyRedirectResponse) {
        $response = new RedirectResponse($response->getTargetUrl(), $response->getStatusCode(), $response->headers->all());
    } else {
        $response = new Response($response->getContent(), $response->getStatusCode(), $response->headers->all());
    }

    return $response->withException($e);
}

```

# laravel 错误处理

```php
public function handleError($level, $message, $file = '', $line = 0, $context = [])
{
    if (error_reporting() & $level) {
        throw new ErrorException($message, 0, $level, $file, $line);
    }
}

public function handleShutdown()
{
    if (! is_null($error = error_get_last()) && $this->isFatal($error['type'])) {
        $this->handleException($this->fatalExceptionFromError($error, 0));
    }
}

protected function fatalExceptionFromError(array $error, $traceOffset = null)
{
    return new FatalErrorException(
        $error['message'], $error['type'], 0, $error['file'], $error['line'], $traceOffset
    );
}

protected function isFatal($type)
{
    return in_array($type, [E_COMPILE_ERROR, E_CORE_ERROR, E_ERROR, E_PARSE]);
}
```

对于不致命的错误，例如 `notice`级别的错误，`handleError` 即可截取， `laravel` 将错误转化为了异常，交给了 `handleException` 去处理。

对于致命错误，例如 `E_PARSE` 解析错误，`handleShutdown` 将会启动，并且判断当前脚本结束是否是由于致命错误，如果是致命错误，将会将其转化为 `FatalErrorException`, 交给了 `handleException` 作为异常去处理。