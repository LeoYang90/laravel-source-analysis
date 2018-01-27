# 前言

当所有的路由都加载完毕后，就会根据请求的 `url` 来将请求分发到对应的路由上去。然而，在分发到路由之前还要经过各种中间件的计算。`laravel` 利用装饰者模式来实现中间件的功能。

# 从原始装饰者模式到闭包装饰者

装饰者模式是设计模式的一种，主要进行对象的多次处理与过滤，是在开放-关闭原则下实现动态添加或减少功能的一种方式。下面先看一个装饰者模式的例子：

总共有两种咖啡：Decaf、Espresso，另有两种调味品：Mocha、Whip（3种设计的主要差别在于抽象方式不同）

装饰模式分为3个部分：

1，抽象组件 -- 对应Coffee类

2，具体组件 -- 对应具体的咖啡，如：Decaf，Espresso

3，装饰者 -- 对应调味品，如：Mocha，Whip

## 原始装饰者模式

```php
public interface Coffee
{  
    public double cost();  
} 

public class Espresso implements Coffee 
{  
    public double cost()
    {  
        return 2.5;  
    }  
}  

public class Dressing implements Coffee 
{  
    private Coffee coffee;  

    public Dressing(Coffee coffee)
    {  
        this.coffee = coffee;  
    }  

    public double cost()
    {  
        return coffee.cost();  
    }  
}  

public class Whip extends Dressing {  
    public Whip(Coffee coffee)
    {  
        super(coffee);  
    }  

    public double cost()
    {  
        return super.cost() + 0.1;  
    }  
} 

public class Mocha extends Dressing 
{  
    public Mocha(Coffee coffee)
    {  
        super(coffee);  
    }  

    public double cost()
    {  
        return super.cost() + 0.5;  
    }  
}
```

当我们使用装饰者模式的时候：

```php
public class Test {  
    public static void main(String[] args) {  
        Coffee coffee = new Espresso();  
        coffee = new Mocha(coffee);  
        coffee = new Mocha(coffee);  
        coffee = new Whip(coffee);  
        //3.6(2.5 + 0.5 + 0.5 + 0.1)  
        System.out.println(coffee.cost());  
    }  
}
```

我们可以看出来，装饰者模式就是利用装饰者类来对具体类不断的进行多层次的处理，首先我们创建了 `Espresso` 类，然后第一次利用 `Mocha` 装饰者对 `Espresso` 咖啡加了摩卡，第二次重复加了摩卡，第三次利用装饰者 `Whip` 对 `Espresso` 咖啡加了奶油。每次加入新的调料，装饰者都会对价格 `cost` 做一些处理（+0.1、+0.5）。

## 无构造函数的装饰者

我们对这个装饰者进行一些改造：

```php
public class Espresso
{  
    double cost;

    public double cost()
    {  
        $this-> cost = 2.5;  
    }  
}  

public class Dressing 
{    
    public double cost(Espresso $espresso)
    {  
        return ($espresso);
    }  
}  

public class Whip extends Dressing 
{  
    public double cost(Espresso $espresso)
    {  
        $espresso->cost = espresso->cost() + 0.1;  

        return ($espresso);
    }  
} 

public class Mocha extends Dressing 
{   
    public double cost(Espresso $espresso)
    {  
        $espresso->cost = espresso->cost() + 0.5;  

        return ($espresso);
    }  
}
```

```php
public class Test {  
    public static void main(String[] args) {  
        Coffee $coffee = new Espresso();  

        $coffee = (new Mocha())->cost($coffee); 
        $coffee = (new Mocha())->cost($coffee); 
        $coffee = (new Whip())->cost($coffee);  

        //3.6(2.5 + 0.5 + 0.5 + 0.1)  
        System.out.println(coffee.cost());  
    }  
}
```

改造后，装饰者类通过函数 `cost` 来注入具体类 `caffee`，而不是通过构造函数，这样做有助于自动化进行装饰处理。我们改造后发现，想要对具体类通过装饰类进行处理，需要不断的调用 `cost` 函数，如果有10个装饰操作，就要手动写10个语句，因此我们继续进行改造：

## 闭包装饰者模式

```php
public class Espresso
{  
    double cost;

    public double cost()
    {  
        $this-> cost = 2.5;  
    }  
}  

public class Dressing 
{    
    public double cost(Espresso $espresso,  Closure $closure)
    {  
        return ($espresso);
    }  
}  

public class Whip extends Dressing 
{  
    public double cost(Espresso $espresso, Closure $closure)
    {  
        $espresso->cost = espresso->cost() + 0.1;  

        return $closure($espresso);
    }  
} 

public class Mocha extends Dressing 
{   
    public double cost(Espresso $espresso, Closure $closure)
    {  
        $espresso->cost = espresso->cost() + 0.5;  

        return $closure($espresso);
    }  
}
```

```php
public class Test {  
    public static void main(String[] args) {  
        Coffee $coffee = new Espresso();  

        $fun = function($coffee，$fuc，$dressing) {
            $dressing->cost($coffee, $fuc); 
        }                


        $fuc0 = function($coffee) {
            return $coffee;
        };

        $fuc1 = function($coffee) use ($fuc0, $dressing = (new Mocha()，$fun)) {
            return $fun($coffee, $fuc0, $dressing);
        }

        $fuc2 = function($coffee) use ($fuc1, $dressing = (new Mocha()，$fun)) {
            return $fuc($coffee, $fun1, $dressing);
        }

        $fuc3 = function($coffee) use ($fuc2, $dressing = (new Whip()，$fun)) {
            return $fuc($coffee, $fun2, $dressing);
        }

        $coffee = $fun3($coffee);

        //3.6(2.5 + 0.5 + 0.5 + 0.1)  
        System.out.println(coffee.cost());  
    }  
}
```

在这次改造中，我们使用了闭包函数，这样做的目的在于，我们只需要最后一句 `$fun3($coffee)`,就可以启动整个装饰链条。

## 闭包装饰者的抽象化

然而这种改造还不够深入，因为我们还可以把 `$fuc1`、`$fuc2`、`$fuc3` 继续抽象化为一个闭包函数，这个闭包函数仅仅是参数 `$fuc`、`$dressing` 每次不同，`$coffee` 相同，因此改造如下：

```php
public class Test {  
    public static void main(String[] args) {  
        Coffee $coffee = new Espresso();  

        $fun = function($coffee) use ($fuc，$dressing) {
            $dressing->cost($coffee, $fuc); 
        }

        $fuc = function($fuc，$dressing) use ($fun) {
            return $fun;
        };


        $fuc0 = function($coffee) {
            return $coffee;
        };

        $fuc1 = $fuc($fuc0, (new Mocha());

        $fuc2 = $fuc($fuc1, (new Mocha());

        $fuc3 = $fuc($fuc2, (new Whip());

        $coffee = $fun3($coffee);

        //3.6(2.5 + 0.5 + 0.5 + 0.1)  
        System.out.println(coffee.cost());  
    }  
}
```

这次，我们把之前的闭包分为两个部分，`$fun` 负责具体类的参数传递，`$fuc`负责装饰者和闭包函数的参数传递。在最后一句 `$fun3`,只需要传递一个具体类，就可以启动整个装饰链条。

## 闭包装饰者的自动化

到这里，我们还有一件事没有完成，那就是 `$fuc1`、`$fuc2`、`$fuc3` 这些闭包的构建还是手动的，我们需要将这个过程改为自动的：

```php
public class Test {  
    public static void main(String[] args) {  
        Coffee $coffee = new Espresso();  

        $fun = function($coffee) use ($fuc，$dressing) {
            $dressing->cost($coffee, $fuc); 
        }

        $fuc = function($fuc，$dressing) use ($fun) {
            return $fun;
        };


        $fuc0 = function($coffee) {
            return $coffee;
        };

        $fucn = array_reduce(
            [(new Mocha(),(new Mocha(),(new Whip()], $fuc, $fuc0
        );

        $coffee = $fucn($coffee);

        //3.6(2.5 + 0.5 + 0.5 + 0.1)  
        System.out.println(coffee.cost());  
    }  
}
```

# laravel的闭包装饰者——Pipeline

上一章我们说到了路由的注册启动与加载过程，这个过程由 `bootstrap()` 完成。当所有的路由加载完毕后，就要进行各种中间件的处理了：

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

public function shouldSkipMiddleware()
{
    return $this->bound('middleware.disable') &&
           $this->make('middleware.disable') === true;
}
```

`laravel` 的中间件处理由 `Pipeline` 来完成，它是一个闭包装饰者模式，其中

* `request` 是具体类，相当于我们上面的 `caffee` 类；
* `middleware` 中间件是装饰者类，相当于上面的 `dressing` 类；

我们先看看这个类内部的代码：

```php
class Pipeline implements PipelineContract
{
    public function __construct(Container $container = null)
    {
      $this->container = $container;
    }

    public function send($passable)
    {
        $this->passable = $passable;

        return $this;
    }

    public function through($pipes)
    {
        $this->pipes = is_array($pipes) ? $pipes : func_get_args();

        return $this;
    }

    public function then(Closure $destination)
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination)
        );

        return $pipeline($this->passable);
    }

    protected function prepareDestination(Closure $destination)
    {
        return function ($passable) use ($destination) {
            return $destination($passable);
        };
    }

    protected function carry()
    {
        return function ($stack, $pipe) {
            return function ($passable) use ($stack, $pipe) {
                if ($pipe instanceof Closure) {
                    return $pipe($passable, $stack);
                } elseif (! is_object($pipe)) {
                    list($name, $parameters) = $this->parsePipeString($pipe);

                    $pipe = $this->getContainer()->make($name);

                    $parameters = array_merge([$passable, $stack], $parameters);
                } else {
                    $parameters = [$passable, $stack];
                }

                return $pipe->{$this->method}(...$parameters);
            };
        };
    }
}
```

`pipeline` 的构造和我们上面所讲的闭包装饰者相同，我们着重来看 `carry()` 函数的代码：

```php
function ($stack, $pipe) {
    ...
}
```

最外层的闭包相当于上个章节的 `$fuc`,

```php
function ($passable) use ($stack, $pipe) {
    ...
}
```

里面的这一层比闭包型党与上个章节的 `$fun`，

`prepareDestination` 这个函数相当于上面的 `$fuc0`,

```php
            if ($pipe instanceof Closure) {
                return $pipe($passable, $stack);
            } elseif (! is_object($pipe)) {
                list($name, $parameters) = $this->parsePipeString($pipe);

                $pipe = $this->getContainer()->make($name);

                $parameters = array_merge([$passable, $stack], $parameters);
            } else {
                $parameters = [$passable, $stack];
            }

            return $pipe->{$this->method}(...$parameters);
```

这一部分相当于上个章节的 `$dressing->cost($coffee, $fuc);`,这部分主要解析中间件 `handle()` 函数的参数：

```php
public function via($method)
{
    $this->method = $method;

    return $this;
}

protected function parsePipeString($pipe)
{
    list($name, $parameters) = array_pad(explode(':', $pipe, 2), 2, []);

    if (is_string($parameters)) {
        $parameters = explode(',', $parameters);
    }

    return [$name, $parameters];
}
```

这样，`laravel` 就实现了中间件对 `request` 的层层处理。

