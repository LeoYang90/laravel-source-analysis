# 前言
在前面几个博客中，我详细讲了 Ioc 容器各个功能的使用、绑定的源码、解析的源码，今天这篇博客会详细介绍 Ioc 容器的一些细节，一些特性，以便更好地掌握容器的功能。

注：本文使用的测试类与测试对象都取自 laravel 的单元测试文件src/illuminate/tests/Container/ContainerTest.php

# rebind绑定特性
## rebind 在绑定之前
instance 和 普通 bind 绑定一样，当重新绑定的时候都会调用 rebind 回调函数，但是有趣的是，对于普通 bind 绑定来说，rebind 回调函数被调用的条件是当前接口被解析过：
```php
public function testReboundListeners()
{
    unset($_SERVER['__test.rebind']);

    $container = new Container;
    $container->rebinding('foo', function () {
        $_SERVER['__test.rebind'] = true;
    });
    $container->bind('foo', function () {
    });
    $container->make('foo');
    $container->bind('foo', function () {
    });

    $this->assertTrue($_SERVER['__test.rebind']);
}
```
所以遇到下面这样的情况,rebinding 的回调函数是不会调用的：
```php
public function testReboundListeners()
{
    unset($_SERVER['__test.rebind']);

    $container = new Container;
    $container->rebinding('foo', function () {
        $_SERVER['__test.rebind'] = true;
    });
    $container->bind('foo', function () {
    });
    $container->bind('foo', function () {
    });

    $this->assertFalse(isset($_SERVER['__test.rebind']));
}
```
有趣的是对于 instance 绑定：
```php
public function testReboundListeners()
{
    unset($_SERVER['__test.rebind']);

    $container = new Container;
    $container->rebinding('foo', function () {
        $_SERVER['__test.rebind'] = true;
    });
    $container->bind('foo', function () {
    });
    $container->instance('foo', function () {
    });

    $this->assertTrue(isset($_SERVER['__test.rebind']));
}
```
rebinding 回调函数却是可以被调用的。其实原因就是 instance 源码中 rebinding 回调函数调用的条件是 rebound 为真，而普通 bind 函数调用 rebinding 回调函数的条件是 resolved 为真. 目前笔者不是很清楚为什么要对 instance 和 bind 区别对待，希望有大牛指导。
## rebind 在绑定之后
为了使得 rebind 回调函数在下一次的绑定中被激活，在 rebind 函数的源码中，如果判断当前对象已经绑定过，那么将会立即解析：
```php
public function rebinding($abstract, Closure $callback)
{
    $this->reboundCallbacks[$abstract = $this->getAlias($abstract)][] = $callback;
    
    if ($this->bound($abstract)) {
        return $this->make($abstract);
    }
}
```
单元测试代码：
```php
public function testReboundListeners1()
{
    unset($_SERVER['__test.rebind']);

    $container = new Container;
    $container->bind('foo', function () {
        return 'foo';
    });

    $container->resolving('foo', function () {
        $_SERVER['__test.rebind'] = true;
    });

    $container->rebinding('foo', function ($container,$object) {//会立即解析
        $container['foobar'] = $object.'bar';
    });

    $this->assertTrue($_SERVER['__test.rebind']);

    $container->bind('foo', function () {
    });

    $this->assertEquals('bar', $container['foobar']);
}
```
# resolving 特性
## resolving 回调的类型
resolving 不仅可以针对接口执行回调函数，还可以针对接口实现的类型进行回调函数。
```php
public function testResolvingCallbacksAreCalledForType()
{
    $container = new Container;
    $container->resolving('StdClass', function ($object) {
        return $object->name = 'taylor';
    });
    $container->bind('foo', function () {
          return new StdClass;
    });
    $instance = $container->make('foo');

    $this->assertEquals('taylor', $instance->name);
}
public function testResolvingCallbacksShouldBeFiredWhenCalledWithAliases()
{
    $container = new Container;
    $container->alias('StdClass', 'std');
    $container->resolving('std', function ($object) {
        return $object->name = 'taylor';
    });
    $container->bind('foo', function () {
        return new StdClass;
    });
    $instance = $container->make('foo');

    $this->assertEquals('taylor', $instance->name);
}
```
## resolving 回调与 instance
前面讲过，对于 singleton 绑定来说，resolving 回调函数仅仅运行一次，只在 singleton 第一次解析的时候才会调用。如果我们利用 instance 直接绑定类的对象，不需要解析，那么 resolving 回调函数将不会被调用：
```php
public function testResolvingCallbacksAreCalledForSpecificAbstracts()
{
    $container = new Container;
    $container->resolving('foo', function ($object) {
        return $object->name = 'taylor';
    });
    $obj = new StdClass;
    $container->instance('foo', $obj);
    $instance = $container->make('foo');

    $this->assertFalse(isset($instance->name));
}
```
# extend 扩展特性
extend 用于扩展绑定对象的功能，对于普通绑定来说，这个函数的位置很灵活：
## 在绑定前扩展
```php
public function testExtendIsLazyInitialized()
{
    ContainerLazyExtendStub::$initialized = false;
    
    $container = new Container;      
    $container->extend('Illuminate\Tests\Container\ContainerLazyExtendStub', function ($obj, $container) {
        $obj->init();
        return $obj;   
    });    
    $container->bind('Illuminate\Tests\Container\ContainerLazyExtendStub'); 

    $this->assertFalse(ContainerLazyExtendStub::$initialized);   
    $container->make('Illuminate\Tests\Container\ContainerLazyExtendStub');   
    $this->assertTrue(ContainerLazyExtendStub::$initialized);
}
```
## 在绑定后解析前扩展
```php
public function testExtendIsLazyInitialized()
{
    ContainerLazyExtendStub::$initialized = false;
    
    $container = new Container;   
    $container->bind('Illuminate\Tests\Container\ContainerLazyExtendStub');    
    $container->extend('Illuminate\Tests\Container\ContainerLazyExtendStub', function ($obj, $container) {
        $obj->init();
        return $obj;   
    });    

    $this->assertFalse(ContainerLazyExtendStub::$initialized);   
    $container->make('Illuminate\Tests\Container\ContainerLazyExtendStub');   
    $this->assertTrue(ContainerLazyExtendStub::$initialized);
}
```
## 在解析后扩展
```php
public function testExtendIsLazyInitialized()
{
    ContainerLazyExtendStub::$initialized = false;
    
    $container = new Container;   
    $container->bind('Illuminate\Tests\Container\ContainerLazyExtendStub');         
    
    $container->make('Illuminate\Tests\Container\ContainerLazyExtendStub'); 
    $this->assertFalse(ContainerLazyExtendStub::$initialized);
    
    $container->extend('Illuminate\Tests\Container\ContainerLazyExtendStub', function ($obj, $container) {
        $obj->init();
        return $obj;   
    });
    $this->assertFalse(ContainerLazyExtendStub::$initialized);  
      
    $container->make('Illuminate\Tests\Container\ContainerLazyExtendStub'); 
    $this->assertTrue(ContainerLazyExtendStub::$initialized);
}
```
可以看出，无论在哪个位置，extend 扩展都有 lazy 初始化的特点，也就是使用 extend 函数并不会立即起作用，而是要等到 make 解析才会激活。
## extend 与 instance 绑定
对于 instance 绑定来说，暂时 extend 的位置需要位于 instance 之后才会起作用，并且会立即起作用，没有 lazy 的特点：
```php
public function testExtendInstancesArePreserved()
{
    $container = new Container;

    $obj = new StdClass;
    $obj->foo = 'foo';
    $container->instance('foo', $obj);
    $container->extend('foo', function ($obj, $container) {
        $obj->bar = 'baz';

        return $obj;
    });

    $this->assertEquals('foo', $container->make('foo')->foo);
    $this->assertEquals('baz', $container->make('foo')->bar);
}
```
## extend 绑定与 rebind 回调
无论扩展对象是 instance 绑定还是 bind 绑定，extend 都会启动 rebind 回调函数：
```php
public function testExtendReBindingInstance()
{
    $_SERVER['_test_rebind'] = false;

    $container = new Container;
    $container->rebinding('foo',function (){
        $_SERVER['_test_rebind'] = true;
    });

    $obj = new StdClass;
    $container->instance('foo',$obj);

    $container->make('foo');

    $container->extend('foo', function ($obj, $container) {
        return $obj;
    });

    this->assertTrue($_SERVER['_test_rebind']);
}

public function testExtendReBinding()
{
    $_SERVER['_test_rebind'] = false;

    $container = new Container;
    $container->rebinding('foo',function (){
        $_SERVER['_test_rebind'] = true;
    });
    $container->bind('foo',function (){
        $obj = new StdClass;

        return $obj;
    });

    $container->make('foo');

    $container->extend('foo', function ($obj, $container) {
        return $obj;
    });

    this->assertFalse($_SERVER['_test_rebind']);
}
```
# contextual 绑定特性
## contextual 在绑定前
contextual 绑定不仅可以与 bind 绑定合作，相互不干扰，还可以与 instance 绑定相互合作。而且 instance 的位置也很灵活，可以在 contextual 绑定前，也可以在contextual 绑定后：
```php
public function testContextualBindingWorksForExistingInstancedBindings()
{
    $container = new Container;

    $container->instance('Illuminate\Tests\Container\IContainerContractStub', new ContainerImplementationStub);

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectOne')->needs('Illuminate\Tests\Container\IContainerContractStub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $this->assertInstanceOf(
             'Illuminate\Tests\Container\ContainerImplementationStubTwo',
             $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
     );
}
```
## contextual 在绑定后
```php
public function testContextualBindingWorksForNewlyInstancedBindings()
{
    $container = new Container;

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectOne')->needs('Illuminate\Tests\Container\IContainerContractStub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $container->instance('Illuminate\Tests\Container\IContainerContractStub', new ContainerImplementationStub);

    $this->assertInstanceOf(
            'Illuminate\Tests\Container\ContainerImplementationStubTwo',
        $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
    );
}
```
## contextual 绑定与别名
contextual 绑定也可以在别名上进行，无论赋予别名的位置是 contextual 的前面还是后面：
```php
public function testContextualBindingDoesntOverrideNonContextualResolution()
{
    $container = new Container;

    $container->instance('stub', new ContainerImplementationStub);
    $container->alias('stub', 'Illuminate\Tests\Container\IContainerContractStub');

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectTwo')->needs('Illuminate\Tests\Container\IContainerContractStub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $this->assertInstanceOf(
            'Illuminate\Tests\Container\ContainerImplementationStubTwo',
            $container->make('Illuminate\Tests\Container\ContainerTestContextInjectTwo')->impl
        );

    $this->assertInstanceOf(
            'Illuminate\Tests\Container\ContainerImplementationStub',
            $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
    );
}

public function testContextualBindingWorksOnNewAliasedBindings()
{
    $container = new Container;

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectOne')->needs('Illuminate\Tests\Container\IContainerContractStub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $container->bind('stub', ContainerImplementationStub::class);
    $container->alias('stub', 'Illuminate\Tests\Container\IContainerContractStub');

    $this->assertInstanceOf(
          'Illuminate\Tests\Container\ContainerImplementationStubTwo',
          $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
    );
}
```
## 争议
目前比较有争议的是下面的情况：
```php
public function testContextualBindingWorksOnExistingAliasedInstances()
{
    $container = new Container;

    $container->alias('Illuminate\Tests\Container\IContainerContractStub', 'stub');
    $container->instance('stub', new ContainerImplementationStub);

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectOne')->needs('stub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $this->assertInstanceOf(
        'Illuminate\Tests\Container\ContainerImplementationStubTwo',
        $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
    ); 
}
```
由于instance的特性，当别名被绑定到其他对象上时，别名 stub 已经失去了与 Illuminate\Tests\Container\IContainerContractStub 之间的关系，因此不能使用 stub 代替作上下文绑定。
但是另一方面：
```php
public function testContextualBindingWorksOnBoundAlias()
{
    $container = new Container;

    $container->alias('Illuminate\Tests\Container\IContainerContractStub', 'stub');
    $container->bind('stub', ContainerImplementationStub::class);

    $container->when('Illuminate\Tests\Container\ContainerTestContextInjectOne')->needs('stub')->give('Illuminate\Tests\Container\ContainerImplementationStubTwo');

    $this->assertInstanceOf(
        'Illuminate\Tests\Container\ContainerImplementationStubTwo',
        $container->make('Illuminate\Tests\Container\ContainerTestContextInjectOne')->impl
    ); 
}
```
代码只是从 instance 绑定改为 bind 绑定，由于 bind 绑定只切断了别名中的 alias 数组的联系，并没有断绝abstractAlias数组的联系，因此这段代码却可以通过，很让人难以理解。本人在给 Taylor Otwell 提出 PR 时，作者原话为“I'm not making any of these changes to the container on a patch release.”。也许，在以后(5.5或以后)版本作者会更新这里的逻辑，我们就可以看看服务容器对别名绑定的态度了，大家也最好不要这样用。
# 服务容器中的闭包函数参数
服务容器中很多函数都有闭包函数，这些闭包函数可以放入特定的参数，在绑定或者解析过程中，这些参数会被服务容器自动带入各种类对象或者服务容器实例。
## bind 闭包参数
```php
public function testAliasesWithArrayOfParameters()
{
    $container = new Container;    
    $container->bind('foo', function ($app, $config) {
        return $config;    
    });    

    $container->alias('foo', 'baz');    
    $this->assertEquals([1, 2, 3], $container->makeWith('baz', [1, 2, 3]));
}
```
## extend 闭包参数
```php
public function testExtendedBindings()
{
    $container = new Container;    
    $container['foo'] = 'foo’;    
    $container->extend('foo', function ($old, $container) {
        return $old.'bar’;    
    });
   
    $this->assertEquals('foobar', $container->make('foo'));
    
    $container = new Container;
    
    $container->singleton('foo', function () {
        return (object) ['name' => 'taylor'];    
    });    
    $container->extend('foo', function ($old, $container) {
        $old->age = 26;
        return $old;    
    });
    
    $result = $container->make('foo');
    $this->assertEquals('taylor', $result->name);    
    $this->assertEquals(26, $result->age);   
    $this->assertSame($result, $container->make('foo'));
}
```
## bindmethod 闭包参数
```php
public function testCallWithBoundMethod()
{
    $container = new Container;
    $container->bindMethod('Illuminate\Tests\Container\ContainerTestCallStub@unresolvable', function ($stub,$container) {
        $container['foo'] = 'foo';
        return $stub->unresolvable('foo', 'bar');
    });
    $result = $container->call('Illuminate\Tests\Container\ContainerTestCallStub@unresolvable');
    $this->assertEquals(['foo', 'bar'], $result);
    $this->assertEquals('foo',$container['foo']);
}
```
## resolve 闭包参数
```php
public function testResolvingCallbacksAreCalledForSpecificAbstracts()
{
     $container = new Container;
     $container->resolving('foo', function ($object，$container) {
         return $object->name = 'taylor';
     });
 
     $container->bind('foo', function () {
        return new StdClass;
     });
     $instance = $container->make('foo');

     $this->assertEquals('taylor', $instance->name);
}
```
## rebinding 闭包参数
```php
public function testReboundListeners()
{
    $container = new Container;
    $container->bind('foo', function () {
        return 'foo';
    });
  
    $container->rebinding('foo', function ($container,$object) {
         $container['bar'] = $object.'bar';
    });
  
    $container->bind('foo', function () {
    });

    $this->assertEquals('bar',$container['foobar']);
}
```