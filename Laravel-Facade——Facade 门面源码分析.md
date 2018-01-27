title: Laravel框架门面Facade源码分析
tags:
  - php
  - 源码
  - laravel
  - facade
  - 实时门面
categories:
  - php
  - laravel
  - facade
  - 源码
author: leoyang
date: 2017-03-19 13:53:00
---
# 前言
这篇文章我们开始讲laravel框架中的门面Facade，什么是门面呢？官方文档：

> &emsp;&emsp;Facades（读音：/fəˈsäd/ ）为应用程序的服务容器中可用的类提供了一个「静态」接口。Laravel 自带了很多 facades ，几乎可以用来访问到 Laravel 中所有的服务。Laravel facades 实际上是服务容器中那些底层类的「静态代理」，相比于传统的静态方法， facades 在提供了简洁且丰富的语法同时，还带来了更好的可测试性和扩展性。

&emsp;&emsp;什么意思呢？首先，我们要知道laravel框架的核心就是个Ioc容器即[服务容器](http://d.laravel-china.org/docs/5.4/container)，功能类似于一个工厂模式，是个高级版的工厂。laravel的其他功能例如路由、缓存、日志、数据库其实都是类似于插件或者零件一样，叫做[服务](http://d.laravel-china.org/docs/5.4/providers)。Ioc容器主要的作用就是生产各种零件，就是提供各个服务。在laravel中，如果我们想要用某个服务，该怎么办呢？最简单的办法就是调用服务容器的make函数，或者利用依赖注入，或者就是今天要讲的门面Facade。门面相对于其他方法来说，最大的特点就是简洁，例如我们经常使用的Router，如果利用服务容器的make：

    ```php
	App::make('router')->get('/', function () {
      return view('welcome');
	});
	```
如果利用门面：

	```php
	Route::get('/', function () {
      return view('welcome');
	});
	```
可以看出代码更加简洁。其实，下面我们就会介绍门面最后调用的函数也是服务容器的make函数。

# Facade的原理
&emsp;&emsp;我们以Route为例，来讲解一下门面Facade的原理与实现。我们先来看Route的门面类：

	```php
	class Route extends Facade
	{
		protected static function getFacadeAccessor()
	    {
	        return 'router';
	    }
	}
    ```
    
    
&emsp;&emsp;很简单吧？其实每个门面类也就是重定义一下getFacadeAccessor函数就行了，这个函数返回服务的唯一名称：router。需要注意的是要确保这个名称可以用服务容器的make函数创建成功(App::make('router'))，原因我们马上就会讲到。
&emsp;&emsp;那么当我们写出Route::get()这样的语句时，到底发生了什么呢？奥秘就在基类Facade中。

	```php
	public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
    ```
&emsp;&emsp;当运行Route::get()时，发现门面Route没有静态get()函数，PHP就会调用这个魔术函数__callStatic。我们看到这个魔术函数做了两件事：获得对象实例，利用对象调用get()函数。首先先看看如何获得对象实例的：

	```php
	public static function getFacadeRoot()
	{
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
    protected static function getFacadeAccessor()
    {
        throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
    }
    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }

        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }

        return static::$resolvedInstance[$name] = static::$app[$name];
    }
    ```
&emsp;&emsp;我们看到基类getFacadeRoot()调用了getFacadeAccessor()，也就是我们的服务重载的函数，如果调用了基类的getFacadeAccessor，就会抛出异常。在我们的例子里getFacadeAccessor()返回了“router”，接下来getFacadeRoot()又调用了resolveFacadeInstance()。在这个函数里重点就是

	```php
	return static::$resolvedInstance[$name] = static::$app[$name];
	```
我们看到，在这里利用了\$app也就是服务容器创建了“router”，创建成功后放入$resolvedInstance作为缓存，以便以后快速加载。
&emsp;&emsp;好了，Facade的原理到这里就讲完了，但是到这里我们有个疑惑，为什么代码中写Route就可以调用Illuminate\Support\Facades\Route呢？这个就是别名的用途了，很多门面都有自己的别名，这样我们就不必在代码里面写use Illuminate\Support\Facades\Route，而是可以直接用Route了。

# 别名Aliases
&emsp;&emsp;为什么我们可以在laravel中全局用Route，而不需要使用use Illuminate\Support\Facades\Route?其实奥秘在于一个PHP函数：[class_alias](http://www.php.net/manual/zh/function.class-alias.php)，它可以为任何类创建别名。laravel在启动的时候为各个门面类调用了class_alias函数，因此不必直接用类名，直接用别名即可。在config文件夹的app文件里面存放着门面与类名的映射：

	```php
	'aliases' => [

        'App' => Illuminate\Support\Facades\App::class,
        'Artisan' => Illuminate\Support\Facades\Artisan::class,
        'Auth' => Illuminate\Support\Facades\Auth::class,
        ...
        ]

	```
&emsp;&emsp;下面我们来看看laravel是如何为门面类创建别名的。

## 启动别名Aliases服务

&emsp;&emsp;说到laravel的启动，我们离不开index.php：

	```php
	require __DIR__.'/../bootstrap/autoload.php';

	$app = require_once __DIR__.'/../bootstrap/app.php';

	$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

	$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
	);
	...
	```
&emsp;&emsp;第一句就是我们前面博客说的composer的自动加载，接下来第二句获取laravel核心的Ioc容器，第三句“制造”出Http请求的内核，第四句是我们这里的关键，这句牵扯很大，laravel里面所有功能服务的注册加载，乃至Http请求的构造与传递都是这一句的功劳。

	```php
	$request = Illuminate\Http\Request::capture()
	```

&emsp;&emsp;这句是laravel通过全局$\_SERVER数组构造一个Http请求的语句，接下来会调用Http的内核函数handle：

	```php
	    public function handle($request)
	    {
	        try {
	            $request->enableHttpMethodParameterOverride();

	            $response = $this->sendRequestThroughRouter($request);
	        } catch (Exception $e) {
	            $this->reportException($e);
	
	            $response = $this->renderException($request, $e);
	        } catch (Throwable $e) {
	            $this->reportException($e = new FatalThrowableError($e));

	            $response = $this->renderException($request, $e);
	        }

	        event(new Events\RequestHandled($request, $response));
	
	        return $response;
	    }
	```
	
&emsp;&emsp;在handle函数方法中enableHttpMethodParameterOverride函数是允许在表单中使用delete、put等类型的请求。我们接着看sendRequestThroughRouter：

	```php
	    protected function sendRequestThroughRouter($request)
	    {
	        $this->app->instance('request', $request);

	        Facade::clearResolvedInstance('request');

	        $this->bootstrap();

	        return (new Pipeline($this->app))
	                    ->send($request)
		                ->through($this->app->shouldSkipMiddleware() ? [] : 
		                          $this->middleware)
	                    ->then($this->dispatchToRouter());
	    }
	```
&emsp;&emsp;前两句是在laravel的Ioc容器设置request请求的对象实例，Facade中清楚request的缓存实例。bootstrap：

	```php
	    public function bootstrap()
	    {
	        if (! $this->app->hasBeenBootstrapped()) {
	            $this->app->bootstrapWith($this->bootstrappers());
	        }
	    }
		
		protected $bootstrappers = [
	        \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
	        \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
	        \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
	        \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
	        \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
	        \Illuminate\Foundation\Bootstrap\BootProviders::class,
    ];
    ```
&emsp;&emsp;$bootstrappers是Http内核里专门用于启动的组件，bootstrap函数中调用Ioc容器的bootstrapWith函数来创建这些组件并利用组件进行启动服务。app->bootstrapWith：

	```php
	    public function bootstrapWith(array $bootstrappers)
	    {
	        $this->hasBeenBootstrapped = true;

	        foreach ($bootstrappers as $bootstrapper) {
	            $this['events']->fire('bootstrapping: '.$bootstrapper, [$this]);

	            $this->make($bootstrapper)->bootstrap($this);

	            $this['events']->fire('bootstrapped: '.$bootstrapper, [$this]);
	        }
	    }
	```
&emsp;&emsp;可以看到bootstrapWith函数也就是利用Ioc容器创建各个启动服务的实例后，回调启动自己的函数bootstrap，在这里我们只看我们Facade的启动组件

	```php
	\Illuminate\Foundation\Bootstrap\RegisterFacades::class
	```
RegisterFacades的bootstrap函数：

	```php
	class RegisterFacades
	{
	    public function bootstrap(Application $app)
	    {
	        Facade::clearResolvedInstances();

	        Facade::setFacadeApplication($app);

	        AliasLoader::getInstance($app->make('config')->get('app.aliases', []))
	                     ->register();
	    }
	}
	```
&emsp;&emsp;可以看出来，bootstrap做了一下几件事：
> 1. 清除了Facade中的缓存
> 2. 设置Facade的Ioc容器
> 3. 获得我们前面讲的config文件夹里面app文件aliases别名映射数组
> 4. 使用aliases实例化初始化AliasLoader
> 5. 调用AliasLoader->register()

	```php
	    public function register()
	    {
	        if (! $this->registered) {
	            $this->prependToLoaderStack();

	            $this->registered = true;
	        }
	    }

	    protected function prependToLoaderStack()
	    {
	        spl_autoload_register([$this, 'load'], true, true);
	    }
	```
&emsp;&emsp;我们可以看出，别名服务的启动关键就是这个spl_autoload_register，这个函数我们应该很熟悉了，在自动加载中这个函数用于解析命名空间，在这里用于解析别名的真正类名。

## 别名Aliases服务

&emsp;&emsp;我们首先来看看被注册到spl_autoload_register的函数，load：

	```php
	    public function load($alias)
	    {
	        if (static::$facadeNamespace && strpos($alias, 
	                                        static::$facadeNamespace) === 0) {
	            $this->loadFacade($alias);

	            return true;
	        }

	        if (isset($this->aliases[$alias])) {
	            return class_alias($this->aliases[$alias], $alias);
	        }
	    }
	```
&emsp;&emsp;这个函数的下面很好理解，就是class_alias利用别名映射数组将别名映射到真正的门面类中去，但是上面这个是什么呢?实际上，这个是laravel5.4版本新出的功能叫做实时门面服务。

## 实时门面服务

&emsp;&emsp;其实门面功能已经很简单了，我们只需要定义一个类继承Facade即可，但是laravel5.4打算更近一步——自动生成门面子类，这就是实时门面。
&emsp;&emsp;实时门面怎么用？看下面的例子：

	```php
	namespace App\Services;

	class PaymentGateway
	{
	    protected $tax;

	    public function __construct(TaxCalculator $tax)
	    {
	        $this->tax = $tax;
	    }
	}
	```
这是一个自定义的类，如果我们想要为这个类定义一个门面，在laravel5.4我们可以这么做：

	```php
	use Facades\ {
	    App\Services\PaymentGateway
	};

	Route::get('/pay/{amount}', function ($amount) {
	    PaymentGateway::pay($amount);
	});
	```
&emsp;&emsp;当然如果你愿意，你还可以在alias数组为门面添加一个别名映射"PaymentGateway" => "use Facades\App\Services\PaymentGateway"，这样就不用写这么长的名字了。
&emsp;&emsp;那么这么做的原理是什么呢？我们接着看源码：

	```php
	protected static $facadeNamespace = 'Facades\\';
	if (static::$facadeNamespace && strpos($alias, static::$facadeNamespace) === 0) {
	   $this->loadFacade($alias);

	   return true;
	}
	```
&emsp;&emsp;如果命名空间是以Facades\\开头的，那么就会调用实时门面的功能，调用loadFacade函数：

	```php
	protected function loadFacade($alias)
    {
        tap($this->ensureFacadeExists($alias), function ($path) {
            require $path;
        });
    }
    ```
&emsp;&emsp;[tap](https://segmentfault.com/a/1190000008447747)是laravel的全局帮助函数，ensureFacadeExists函数负责自动生成门面类，loadFacade负责加载门面类：

       ```php
        protected function ensureFacadeExists($alias)
	    {
	        if (file_exists($path = storage_path('framework/cache/facade-'.sha1($alias).'.php'))) {
	            return $path;
	        }

	        file_put_contents($path, $this->formatFacadeStub(
	            $alias, file_get_contents(__DIR__.'/stubs/facade.stub')
	        ));

	        return $path;
	    }
	    ```
&emsp;&emsp;可以看出来，laravel框架生成的门面类会放到stroge/framework/cache/文件夹下，名字以facade开头，以命名空间的哈希结尾。如果存在这个文件就会返回，否则就要利用file_put_contents生成这个文件，formatFacadeStub：

        ```php
        protected function formatFacadeStub($alias, $stub)
	    {
	        $replacements = [
	            str_replace('/', '\\', dirname(str_replace('\\', '/', $alias))),
	            class_basename($alias),
	            substr($alias, strlen(static::$facadeNamespace)),
	        ];

	        return str_replace(
	            ['DummyNamespace', 'DummyClass', 'DummyTarget'], $replacements, $stub
	        );
	    }
	```
简单的说，对于Facades\App\Services\PaymentGateway，$replacements第一项是门面命名空间，将Facades\App\Services\PaymentGateway转为Facades/App/Services/PaymentGateway，取前面Facades/App/Services/，再转为命名空间Facades\App\Services\；第二项是门面类名，PaymentGateway；第三项是门面类的服务对象，App\Services\PaymentGateway，用这些来替换门面的模板文件：

	```php
	<?php

	namespace DummyNamespace;

	use Illuminate\Support\Facades\Facade;

	/**
	 * @see \DummyTarget
	 */
	class DummyClass extends Facade
	{
	    /**
	     * Get the registered name of the component.
	     *
	     * @return string
	     */
	    protected static function getFacadeAccessor()
	    {
	        return 'DummyTarget';
	    }
	}
	```
替换后的文件是：

	```php
	<?php

	namespace Facades\App\Services\;

	use Illuminate\Support\Facades\Facade;

	/**
	 * @see \DummyTarget
	 */
	class PaymentGateway extends Facade
	{
	    /**
	     * Get the registered name of the component.
	     *
	     * @return string
	     */
	    protected static function getFacadeAccessor()
	    {
	        return 'App\Services\PaymentGateway';
	    }
	}
	```
就是这么简单！！！


# 结语
&emsp;&emsp;门面的原理就是这些，相对来说门面服务的原理比较简单，和自动加载相互配合使得代码更加简洁，希望大家可以更好的使用这些门面！