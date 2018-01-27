# 服务容器的绑定
## bind 绑定
欢迎关注我的博客：[www.leoyang90.cn](http://www.leoyang90.cn)

bind 绑定是服务容器最常用的绑定方式，在 [上一篇](http://leoyang90.cn/2017/05/06/Laravel-container/)文章中我们讨论过，bind 的绑定有三种：
> - 绑定自身
> - 绑定闭包
> - 绑定接口

今天，我们这篇文章主要从源码上讲解 Ioc 服务容器是如何进行绑定的。
```php
/**
* Register a binding with the container.
*
* @param string|array $abstract
* @param \Closure|string|null $concrete
* @param bool $shared
* @return void
*/
public function bind($abstract, $concrete = null, $shared = false)
{
  // If no concrete type was given, we will simply set the concrete type to the
  // abstract type. After that, the concrete type to be registered as shared
  // without being forced to state their classes in both of the parameters.
  $this->dropStaleInstances($abstract);

  if (is_null($concrete)) {
    $concrete = $abstract;
  }

  // If the factory is not a Closure, it means it is just a class name which is
  // bound into this container to the abstract type and we will just wrap it
  // up inside its own Closure to give us more convenience when extending.
  if (! $concrete instanceof Closure) {
    $concrete = $this->getClosure($abstract, $concrete);
  }

  $this->bindings[$abstract] = compact('concrete', 'shared');

  // If the abstract type was already resolved in this container we'll fire the
  // rebound listener so that any objects which have already gotten resolved
  // can have their copy of the object updated via the listener callbacks.
  if ($this->resolved($abstract)) {
    $this->rebound($abstract);
  }
}
```
从源码中我们可以看出，服务器的绑定有如下几个步骤：
> 1. 去除原有注册。去除当前绑定接口的原有实现单例对象，和原有的别名，为实现绑定新的实现做准备。
> 2. 加装闭包。如果实现类不是闭包（绑定自身或者绑定接口），那么就创建闭包，以实现 lazy 加载。
> 3. 注册。将闭包函数和单例变量存入 bindings 数组中，以备解析时使用。
> 4. 回调。如果绑定的接口已经被解析过了，将会调用回调函数，对已经解析过的对象进行调整。

### 去除原有注册 
dropStaleInstances 用于去除当前接口原有的注册和别名，这里负责清除绑定的 aliases 和单例对象的 instances，bindings 后面再做修改：
```php
protected function dropStaleInstances($abstract)
{
    unset($this->instances[$abstract], $this->aliases[$abstract]);
}
```
### 加装闭包
getClosure 的作用是为注册的非闭包实现外加闭包，这样做有两个作用：

- 延时加载
  
服务容器在 getClosure 中为每个绑定的类都包一层闭包，这样服务容器就只有进行解析的时候闭包才会真正进行运行，实现了 lazy 加载的功能。
- 递归绑定
  
对于服务容器来说，绑定是可以递归的，例如：
```php
$app->bind(A::class,B::class);
$app->bind(B::class,C::class);
$app->bind(C::class,function(){
    return new C;
})
```
对于 A 类，我们直接解析 A 可以得到 B 类，但是如果仅仅到此为止，服务容器直接去用反射去创建 B 类的话，那么就很有可能创建失败，因为 B 类很有可能也是接口，B 接口绑定了其他实现类，要知道接口是无法实例化的。

因此服务容器需要递归地对 A 进行解析，这个就是 getClosure 的作用，它把所有可能会递归的绑定在闭包中都用 make 函数，这样解析 make(A::class) 的时候得到闭包 make(B::class)，make(B::class) 的时候会得到闭包 make(C::class)，make(C::class) 终于可以得到真正的实现了。

对于自我绑定的情况，因为不存在递归情况，所以在闭包中会使用 build 函数直接创建对象。（如果仍然使用 make，那就无限循环了）
  
```php
protected function getClosure($abstract, $concrete)
{
    return function ($container, $parameters = []) use ($abstract, $concrete) {
        if ($abstract == $concrete) {
            return $container->build($concrete);
        }
        return $container->makeWith($concrete, $parameters);
    };
}
```
### 注册
注册就是向 binding 数组中添加注册的接口与它的实现，其中 compact() 函数创建包含变量名和它们的值的数组，创建后的结果为：
```php
$bindings[$abstract] = [
  'concrete' => $concrete,
  'shared' => $shared
]
```
### 回调
注册之后，还要查看当前注册的接口是否已经被实例化，如果已经被服务容器实例化过，那么就要调用回调函数。（若存在回调函数）
resolved() 函数用于判断当前接口是否曾被解析过，在判断之前，先获取了接口的最终服务名：
```php
public function resolved($abstract)
{
    if ($this->isAlias($abstract)) {
        $abstract = $this->getAlias($abstract);
    }

    return isset($this->resolved[$abstract]) ||
        isset($this->instances[$abstract]);
}
    
public function isAlias($name)
{
    return isset($this->aliases[$name]);
}
```
getAlias() 函数利用递归的方法获取别名的最终服务名称：

```php
public function getAlias($abstract)
{
    if (! isset($this->aliases[$abstract])) {
        return $abstract;
    }

    if ($this->aliases[$abstract] === $abstract) {
        throw new LogicException("[{$abstract}] is aliased to itself.");
    }

    return $this->getAlias($this->aliases[$abstract]);
}
```
如果当前接口已经被解析过了，那么就要运行回调函数：
```php
protected function rebound($abstract)
{
    $instance = $this->make($abstract);

    foreach ($this->getReboundCallbacks($abstract) as $callback) {
        call_user_func($callback, $this, $instance);
    }
}
    
protected function getReboundCallbacks($abstract)
{
    if (isset($this->reboundCallbacks[$abstract])) {
        return $this->reboundCallbacks[$abstract];
    }

    return [];
}
```
这里面的 reboundCallbacks 从哪里来呢？这就是 [Laravel核心——Ioc服务容器](http://www.leoyang90.cn/2017/05/06/Laravel-container/)  文章中提到的 rebinding
```php
public function rebinding($abstract, Closure $callback)
{
    $this->reboundCallbacks[$abstract = $this->getAlias($abstract)][] = $callback;
   
    if ($this->bound($abstract)) {
        return $this->make($abstract);
    }
}
```
值得注意的是： rebinding 函数不仅绑定了回调函数，同时顺带还对接口abstract进行了解析，因为只有解析过，下次注册才会调用回调函数。
## singleton 绑定
singleton 绑定仅仅是 bind 绑定的一个 shared 为真的形式：
```php
public function singleton($abstract, $concrete = null)
{
    $this->bind($abstract, $concrete, true);
}
```
## instance 绑定
不对接口进行解析，直接给接口一个实例作为单例对象。从下面可以看出，主要的工作就是去除接口在abstractAliases 数组和 aliases 数组中的痕迹，防止 make 函数根据别名继续解析下去出现错误。如果当前接口曾经注册过，那么就调用回调函数。
```php
public function instance($abstract, $instance)
{
    $this->removeAbstractAlias($abstract);

    $isBound = $this->bound($abstract);
     
    unset($this->aliases[$abstract]); 
           
    $this->instances[$abstract] = $instance;
        
    if ($isBound) {
        $this->rebound($abstract);
    }
}

protected function removeAbstractAlias($searched)
{
    if (! isset($this->aliases[$searched])) {
       return;
    }

    foreach ($this->abstractAliases as $abstract => $aliases) {
        foreach ($aliases as $index => $alias) {
            if ($alias == $searched) {
                unset($this->abstractAliases[$abstract][$index]);
            }
        }
    }
}

public function bound($abstract)
{
    return isset($this->bindings[$abstract]) ||
           isset($this->instances[$abstract]) ||
           $this->isAlias($abstract);
}
```
## Context 绑定
Context 绑定一般用于依赖注入，当我们利用依赖注入来自动实例化对象时，服务容器其实是利用反射机制来为构造函数实例化它的参数，这个过程中，被实例化的对象就是下面的 concrete，构造函数的参数接口是 abstract，参数接口实际的实现是 implementation。
例如：
```php
$this->app->when(PhotoController::class)
          ->needs(Filesystem::class)
          ->give(function () {
              return Storage::disk('local');
          });
```
这里实例化对象 concrete 就是 PhotoController，构造函数的参数接口 abstract 就是 Filesystem。参数接口实际实现 implementation 是 Storage::disk('local')。

这样，每次进行解析构造函数的参数接口的时候，都会去判断当前的 contextual 数组里面 concrete[concrete] [abstract]（也就是 concrete[PhotoController::class] [Filesystem::class]）对应的上下文绑定，如果有就直接从数组中取出来，如果没有就按照正常方式解析。
值得注意的是，concrete 和 abstract 都是利用 getAlias 函数，保证最后拿到的不是别名。
```php
public function when($concrete)
{
    return new ContextualBindingBuilder($this, $this->getAlias($concrete));
}
public function __construct(Container $container, $concrete)
{
    $this->concrete = $concrete;
    $this->container = $container;
}
public function needs($abstract)
{
    $this->needs = $abstract;

    return $this;
}
public function give($implementation)
{
    $this->container->addContextualBinding(
        $this->concrete, $this->needs, $implementation
    );
}
public function addContextualBinding($concrete, $abstract, $implementation)
{
    $this->contextual[$concrete][$this->getAlias($abstract)] = $implementation;
}
```
## tag 绑定
标签绑定比较简单，绑定过程就是将标签和接口之间建立一个对应数组，在解析的过程中，按照标签把所有接口都解析一遍即可。
```php
public function tag($abstracts, $tags)
{
    $tags = is_array($tags) ? $tags : array_slice(func_get_args(), 1);

    foreach ($tags as $tag) {
        if (! isset($this->tags[$tag])) {
            $this->tags[$tag] = [];
        }

        foreach ((array) $abstracts as $abstract) {
            $this->tags[$tag][] = $abstract;
        }
    }
}
```
## 数组绑定
利用数组进行绑定的时候 ($app()[A::class] = B::class)，服务容器会调用 offsetSet 函数:
```php
public function offsetSet($key, $value)
{
    $this->bind($key, $value instanceof Closure ? $value : function () use ($value) {
            return $value;
    });
}
```
# extend扩展
extend 扩展分为两种，一种是针对instance注册的对象，这种情况将立即起作用，并更新之前实例化的对象;另一种情况是非 instance 注册的对象，那么闭包函数将会被放入 extenders 数组中，将在下一次实例化对象的时候才起作用：
```php
public function extend($abstract, Closure $closure)
{
    $abstract = $this->getAlias($abstract);

    if (isset($this->instances[$abstract])) {
        $this->instances[$abstract] = $closure($this->instances[$abstract], $this);

        $this->rebound($abstract);
    } else {
        $this->extenders[$abstract][] = $closure;
        
        if ($this->resolved()) {
            $this->rebound($abstract);
        }
    }
}
```
# 服务器事件
服务器的事件注册依靠 resolving 函数和 afterResolving 函数，这两个函数维护着 globalResolvingCallbacks、resolvingCallbacks、globalAfterResolvingCallbacks、afterResolvingCallbacks 数组，这些数组中存放着事件的回调闭包函数，每当对对象进行解析时就会遍历这些数组，触发事件：
```php
public function resolving($abstract, Closure $callback = null)
{
    if (is_string($abstract)) {
        $abstract = $this->getAlias($abstract);
    }

    if (is_null($callback) && $abstract instanceof Closure) {
        $this->globalResolvingCallbacks[] = $abstract;
    } else {
        $this->resolvingCallbacks[$abstract][] = $callback;
    }
}

public function afterResolving($abstract, Closure $callback = null)
{
    if (is_string($abstract)) {
        $abstract = $this->getAlias($abstract);
    }

    if ($abstract instanceof Closure && is_null($callback)) {
        $this->globalAfterResolvingCallbacks[] = $abstract;
    } else {
        $this->afterResolvingCallbacks[$abstract][] = $callback;
    }
}
```
> Written with [StackEdit](https://stackedit.io/).