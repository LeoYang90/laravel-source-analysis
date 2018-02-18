## 前言

[上一篇文章](http://leoyang90.cn/2017/03/11/PHP-Composer-autoload/)中，我们讨论了 `PHP自动加载功能` 、`PHP命名空间`、`PSR0/PSR4标准`，有了这些知识，其实我们就可以按照 PSR4标准 写出可以自动加载的程序了。然而我们为什么要自己写呢？尤其是有 `Composer` 这神一样的包管理器的情况下？

# Composer自动加载概论

## 简介
`Composer` 是 PHP 的一个依赖管理工具。它允许你申明项目所依赖的代码库，它会在你的项目中为你安装他们。详细内容可以查看[Composer 中文网](http://docs.phpcomposer.com/00-intro.html)。

#### Composer Composer 将这样为你解决问题：

> - 你有一个项目依赖于若干个库。
> - 其中一些库依赖于其他库。
> - 你声明你所依赖的东西。
> - Composer 会找出哪个版本的包需要安装，并安装它们（将它们下载到你的项目中）。

例如，你正在创建一个项目，你需要一个库来做日志记录。你决定使用 `monolog` 。为了将它添加到你的项目中，你所需要做的就是创建一个 `composer.json` 文件，其中描述了项目的依赖关系。

```php
 {
   "require": {
     "monolog/monolog": "1.2.*"
   }
 }
```
然后我们只要在项目里面直接`use Monolog\Logger`即可，神奇吧！

简单的说，Composer 帮助我们下载好了符合 `PSR0/PSR4标准` 的第三方库，并把文件放在相应位置；帮我们写了 `__autoload()` 函数，注册到了 `spl_register()` 函数，当我们想用第三方库的时候直接使用命名空间即可。

那么当我们想要写自己的命名空间的时候，该怎么办呢？很简单，我们只要按照 PSR4标准 命名我们的命名空间，放置我们的文件，然后在 composer 里面写好顶级域名与具体目录的映射，就可以享用 composer 的便利了。

当然如果有一个非常棒的框架，我们会惊喜地发现，在 composer 里面写顶级域名映射这事我们也不用做了，框架已经帮我们写好了顶级域名映射了，我们只需要在框架里面新建文件，在新建的文件中写好命名空间，就可以在任何地方 use 我们的命名空间了。

下面我们就以 Laravel 框架为例，讲一讲 composer 是如何实现 `PSR0/PSR4标准` 的自动加载功能。

## Composer自动加载文件

&emsp;&emsp;首先，我们先大致了解一下Composer自动加载所用到的源文件。

> 1. autoload_real.php: 自动加载功能的引导类。
>    * 任务是composer加载类的初始化`(顶级命名空间与文件路径映射初始化)`和注册(spl_autoload_register())。
> 2. ClassLoader.php: composer加载类。
>    * composer自动加载功能的核心类。
> 3. autoload_static.php: 顶级命名空间初始化类，
>    * 用于给核心类初始化顶级命名空间。
> 4. autoload_classmap.php: 自动加载的最简单形式，
>    * 有完整的命名空间和文件目录的映射；
> 5. autoload_files.php: 用于加载全局函数的文件，
>    * 存放各个全局函数所在的文件路径名；
> 6. autoload_namespaces.php: 符合PSR0标准的自动加载文件，
>    * 存放着顶级命名空间与文件的映射；
> 7. autoload_psr4.php: 符合PSR4标准的自动加载文件，
>    * 存放着顶级命名空间与文件的映射；

# laravel框架下Composer的自动加载源码分析——启动
---
&emsp;&emsp; laravel框架的初始化是需要composer自动加载协助的，所以laravel的入口文件index.php第一句就是利用composer来实现自动加载功能。

```php
<?php
   require __DIR__.'/../bootstrap/autoload.php';
```
咱们接着去看 bootstrap 目录下的 `autoload.php` ：

```php
<?php
  define('LARAVEL_START', microtime(true));

  require __DIR__ . '/../vendor/autoload.php';
```

再去 vendor 目录下的 `autoload.php` ：

```php
<?php
  require_once __DIR__ . '/composer' . '/autoload_real.php';

  return ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e::getLoader();
```

为什么框架要在 `bootstrap/autoload.php` 转一下？个人理解，laravel 这样设计有利于支持或扩展**任意**有自动加载的第三方库。

好了，我们终于要看到了Composer真正要显威的地方了。`autoload_real.php`里面就是一个自动加载功能的引导类，这个类不负责具体功能逻辑，只做了两件事：初始化自动加载类、注册自动加载类。

到autoload_real这个文件里面去看，发现这个引导类的名字叫ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e，为什么要叫这么古怪的名字呢？因为这是防止用户自定义类名跟这个类重复冲突了，所以在类名上加了一个hash值。其实还有一个做法我们更加熟悉，那就是不直接定义类名，而是定义一个命名空间。这里为什么不定义一个命名空间呢？个人理解：命名空间一般都是为了复用，而这个类只需要运行一次即可，以后也不会用得到，用hash值更加合适。

# laravel框架下Composer的自动加载源码分析——autoload_real引导类
---
在 vendor 目录下的 `autoload.php` 文件中我们可以看出，程序主要调用了引导类的静态方法 `getLoader()` ，我们接着看看这个函数。

```php
<?php
	public static function getLoader()
	{
	  /***************************经典单例模式********************/
	  if (null !== self::$loader) {
	      return self::$loader;
	  }

	  /***********************获得自动加载核心类对象********************/
	  spl_autoload_register(
        array('ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader'), true, true
      );

	  self::$loader = $loader = new \Composer\Autoload\ClassLoader();

	  spl_autoload_unregister(
        array('ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader')
      );

	  /***********************初始化自动加载核心类对象********************/
	  $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION');

	  if ($useStaticLoader) {
	      require_once __DIR__ . '/autoload_static.php';

	      call_user_func(
          \Composer\Autoload\ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::getInitializer($loader)
          );

	  } else {
	      $map = require __DIR__ . '/autoload_namespaces.php';
	      foreach ($map as $namespace => $path) {
	          $loader->set($namespace, $path);
	      }

	      $map = require __DIR__ . '/autoload_psr4.php';
	      foreach ($map as $namespace => $path) {
	          $loader->setPsr4($namespace, $path);
	      }

	      $classMap = require __DIR__ . '/autoload_classmap.php';
	      if ($classMap) {
	          $loader->addClassMap($classMap);
	      }
	  }

	  /***********************注册自动加载核心类对象********************/
	  $loader->register(true);

	  /***********************自动加载全局函数********************/
	  if ($useStaticLoader) {
	      $includeFiles = Composer\Autoload\ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$files;
	  } else {
	      $includeFiles = require __DIR__ . '/autoload_files.php';
	  }

	  foreach ($includeFiles as $fileIdentifier => $file) {
	      composerRequire832ea71bfb9a4128da8660baedaac82e($fileIdentifier, $file);
	  }

	  return $loader;
	}
```
  &emsp;&emsp;从上面可以看出，我把自动加载引导类分为5个部分。

## 第一部分——单例

第一部分很简单，就是个最经典的单例模式，自动加载类只能有一个。

```php
<?php
  if (null !== self::$loader) {
      return self::$loader;
  }
```


## 第二部分——构造ClassLoader核心类

第二部分 new 一个自动加载的核心类对象。

```php
<?php
  /***********************获得自动加载核心类对象********************/
  spl_autoload_register(
    array('ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader'), true, true
  );

  self::$loader = $loader = new \Composer\Autoload\ClassLoader();

  spl_autoload_unregister(
    array('ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader')
  );
```

`loadClassLoader()`函数：

```php
<?php
public static function loadClassLoader($class)
{
    if ('Composer\Autoload\ClassLoader' === $class) {
        require __DIR__ . '/ClassLoader.php';
    }
}
```

从程序里面我们可以看出，composer 先向 PHP 自动加载机制注册了一个函数，这个函数 require 了 ClassLoader 文件。成功 new 出该文件中核心类 ClassLoader() 后，又销毁了该函数。

为什么不直接require，而要这么麻烦？原因就是怕有的用户也定义了个 `\Composer\Autoload\ClassLoader` 命名空间，导致自动加载错误文件。那为什么不跟引导类一样用个 hash 呢？因为这个类是可以复用的，框架允许用户使用这个类。

## 第三部分 —— 初始化核心类对象

```php
<?php
  /***********************初始化自动加载核心类对象********************/
  $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION');
  if ($useStaticLoader) {
     require_once __DIR__ . '/autoload_static.php';

     call_user_func(
       \Composer\Autoload\ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::getInitializer($loader)
     );
  } else {
      $map = require __DIR__ . '/autoload_namespaces.php';
      foreach ($map as $namespace => $path) {
         $loader->set($namespace, $path);
      }

      $map = require __DIR__ . '/autoload_psr4.php';
      foreach ($map as $namespace => $path) {
         $loader->setPsr4($namespace, $path);
      }

      $classMap = require __DIR__ . '/autoload_classmap.php';
      if ($classMap) {
          $loader->addClassMap($classMap);
      }
    }
 ```
   这一部分就是对自动加载类的初始化，主要是给自动加载核心类初始化顶级命名空间映射。

   初始化的方法有两种：
      1. 使用autoload_static进行静态初始化；
      2. 调用核心类接口初始化。

### autoload_static静态初始化

静态初始化只支持 PHP5.6 以上版本并且不支持 HHVM 虚拟机。我们深入 `autoload_static.php` 这个文件发现这个文件定义了一个用于静态初始化的类，名字叫 `ComposerStaticInit832ea71bfb9a4128da8660baedaac82e`，仍然为了避免冲突加了 hash 值。这个类很简单：


```php
<?php
  class ComposerStaticInit832ea71bfb9a4128da8660baedaac82e{
     public static $files = array(...);
     public static $prefixLengthsPsr4 = array(...);
     public static $prefixDirsPsr4 = array(...);
     public static $prefixesPsr0 = array(...);
     public static $classMap = array (...);

    public static function getInitializer(ClassLoader $loader)
    {
      return \Closure::bind(function () use ($loader) {
          $loader->prefixLengthsPsr4
  						= ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$prefixLengthsPsr4;

          $loader->prefixDirsPsr4
  						= ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$prefixDirsPsr4;

          $loader->prefixesPsr0
  						= ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$prefixesPsr0;

          $loader->classMap
  						= ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$classMap;

      }, null, ClassLoader::class);
  }
```
这个静态初始化类的核心就是 `getInitializer()` 函数，它将自己类中的顶级命名空间映射给了 ClassLoader类。值得注意的是这个函数返回的是一个匿名函数，为什么呢？原因就是 `ClassLoader类` 中的 `prefixLengthsPsr4` 、`prefixDirsPsr4`等等方法都是private的。。。普通的函数没办法给类的 `private` 成员变量赋值。利用匿名函数的绑定功能就可以将把匿名函数转为 ClassLoader类 的成员函数。

关于匿名函数的[绑定功能](http://www.cnblogs.com/yjf512/p/4421289.html)。

接下来就是顶级命名空间初始化的关键了。


 #### 最简单的 classMap:

```php
<?php
  public static $classMap = array (
  	'App\\Console\\Kernel'
  			=> __DIR__ . '/../..' . '/app/Console/Kernel.php',

  	'App\\Exceptions\\Handler'
  			=> __DIR__ . '/../..' . '/app/Exceptions/Handler.php',

  	'App\\Http\\Controllers\\Auth\\ForgotPasswordController'
  			=> __DIR__ . '/../..' . '/app/Http/Controllers/Auth/ForgotPasswordController.php',

  	'App\\Http\\Controllers\\Auth\\LoginController'
  			=> __DIR__ . '/../..' . '/app/Http/Controllers/Auth/LoginController.php',

  	'App\\Http\\Controllers\\Auth\\RegisterController'
  			=> __DIR__ . '/../..' . '/app/Http/Controllers/Auth/RegisterController.php',
  ...)
```

简单吧，直接命名空间全名与目录的映射，没有顶级命名空间。。。简单粗暴，也导致这个数组相当的大。

#### PSR0 顶级命名空间映射：


```php
<?php
  public static $prefixesPsr0 = array (
    'P' => array (
        'Prophecy\\' => array (
            0 => __DIR__ . '/..' . '/phpspec/prophecy/src',
        ),

        'Parsedown' => array (
            0 => __DIR__ . '/..' . '/erusev/parsedown',
        ),
    ),
    'M' => array (
        'Mockery' => array (
            0 => __DIR__ . '/..' . '/mockery/mockery/library',
        ),
    ),
    'J' => array (
        'JakubOnderka\\PhpConsoleHighlighter' => array (
            0 => __DIR__ . '/..' . '/jakub-onderka/php-console-highlighter/src',
        ),
        'JakubOnderka\\PhpConsoleColor' => array (
            0 => __DIR__ . '/..' . '/jakub-onderka/php-console-color/src',
        ),
    ),
    'D' => array (
        'Doctrine\\Common\\Inflector\\' => array (
            0 => __DIR__ . '/..' . '/doctrine/inflector/lib',
        ),
    ),
  );
```

为了快速找到顶级命名空间，我们这里使用命名空间第一个字母作为前缀索引。这个映射的用法比较明显，假如我们有Parsedown/example这样的命名空间，首先通过首字母P，找到

```php
<?php
  'P' => array (
      'Prophecy\\' => array (
		0 => __DIR__ . '/..' . '/phpspec/prophecy/src',
      ),

      'Parsedown' => array (
		0 => __DIR__ . '/..' . '/erusev/parsedown',
      ),
  ),
```


这个数组，然后我们就会遍历这个数组来和 `Parsedown/example` 比较，发现第一个 Prophecy 不符合，第二个 Parsedown 符合，然后得到了映射目录：**(映射目录可能不止一个)**

```php
<?php
	array (0 => __DIR__ . '/..' . '/erusev/parsedown',)
```

我们会接着遍历这个数组，尝试 `__DIR__ . '/..' . '/erusev/parsedown/Parsedown/example.php' ` 是否存在，如果不存在接着遍历数组(这个例子数组只有一个元素)，如果数组遍历完都没有，就会加载失败。

#### PSR4标准顶级命名空间映射数组：

```php
<?php
  public static $prefixLengthsPsr4 = array(
  	'p' => array (
		'phpDocumentor\\Reflection\\' => 25,
	),
  	'S' => array (
		'Symfony\\Polyfill\\Mbstring\\' => 26,
		'Symfony\\Component\\Yaml\\' => 23,
		'Symfony\\Component\\VarDumper\\' => 28,
		...
	),
  ...);

  public static $prefixDirsPsr4 = array (
  	'phpDocumentor\\Reflection\\' => array (
		0 => __DIR__ . '/..' . '/phpdocumentor/reflection-common/src',
		1 => __DIR__ . '/..' . '/phpdocumentor/type-resolver/src',
		2 => __DIR__ . '/..' . '/phpdocumentor/reflection-docblock/src',
	),
  	 'Symfony\\Polyfill\\Mbstring\\' => array (
		0 => __DIR__ . '/..' . '/symfony/polyfill-mbstring',
	),
  	'Symfony\\Component\\Yaml\\' => array (
		0 => __DIR__ . '/..' . '/symfony/yaml',
	),
  ...)
```
PSR4标准顶级命名空间映射用了两个数组，第一个和 PSR0 一样用命名空间第一个字母作为前缀索引，然后是 顶级命名空间，但是最终并不是文件路径，而是 顶级命名空间 的长度。为什么呢？因为前一篇[文章](http://leoyang90.cn/2017/03/11/PHP-Composer-autoload/)我们说过，PSR4标准 的文件目录更加灵活，更加简洁。

PSR0 中顶级命名空间目录直接加到命名空间前面就可以得到路径

    Parsedown/example => __DIR__ . '/..' . '/erusev/parsedown/Parsedown/example.php
                                                               ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
而 PSR4标准 却是用顶级命名空间目录替换顶级命名空间，所以获得顶级命名空间的长度很重要。

    Parsedown/example => __DIR__ . '/..' . '/erusev/parsedown/example.php
                                                    ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑

具体的用法：假如我们找 `Symfony\\Polyfill\\Mbstring\\example` 这个命名空间，和 PSR0 一样通过前缀索引和字符串匹配我们得到了

```php
<?php
	'Symfony\\Polyfill\\Mbstring\\' => 26,
```
这条记录，键是顶级命名空间，值是命名空间的长度。拿到顶级命名空间后去 `$prefixDirsPsr4数组` 获取它的映射目录数组：**(注意映射目录可能不止一条)**

```php
<?php
  'Symfony\\Polyfill\\Mbstring\\' => array (
  	        0 => __DIR__ . '/..' . '/symfony/polyfill-mbstring',
  	    )
```

然后我们就可以将命名空间 `Symfony\\Polyfill\\Mbstring\\example` 前26个字符替换成目录 `__DIR__ . '/..' . '/symfony/polyfill-mbstring` ，我们就得到了`__DIR__ . '/..' . '/symfony/polyfill-mbstring/example.php`，先验证磁盘上这个文件是否存在，如果不存在接着遍历。如果遍历后没有找到，则加载失败。

**自动加载核心类ClassLoader的静态初始化完成！！！**

### ClassLoader接口初始化
---
如果PHP版本低于5.6或者使用 HHVM 虚拟机环境，那么就要使用核心类的接口进行初始化。

```php
<?php
	//PSR0标准
	$map = require __DIR__ . '/autoload_namespaces.php';
	foreach ($map as $namespace => $path) {
	   $loader->set($namespace, $path);
	}

	//PSR4标准
	$map = require __DIR__ . '/autoload_psr4.php';
	foreach ($map as $namespace => $path) {
	   $loader->setPsr4($namespace, $path);
	}

	$classMap = require __DIR__ . '/autoload_classmap.php';
	if ($classMap) {
	   $loader->addClassMap($classMap);
	}
```

 #### **PSR0标准**

autoload_namespaces：


```php
<?php
	return array(
		'Prophecy\\'
			=> array($vendorDir . '/phpspec/prophecy/src'),

		'Parsedown'
			=> array($vendorDir . '/erusev/parsedown'),

		'Mockery'
			=> array($vendorDir . '/mockery/mockery/library'),

		'JakubOnderka\\PhpConsoleHighlighter'
			=> array($vendorDir . '/jakub-onderka/php-console-highlighter/src'),

		'JakubOnderka\\PhpConsoleColor'
			=> array($vendorDir . '/jakub-onderka/php-console-color/src'),

		'Doctrine\\Common\\Inflector\\'
			=> array($vendorDir . '/doctrine/inflector/lib'),
	);
```

PSR0标准的初始化接口：    


```php
<?php
	public function set($prefix, $paths)
	{
	    if (!$prefix) {
	        $this->fallbackDirsPsr0 = (array) $paths;
	    } else {
	        $this->prefixesPsr0[$prefix[0]][$prefix] = (array) $paths;
	    }
	}
```
很简单，PSR0标准取出命名空间的第一个字母作为索引，一个索引对应多个顶级命名空间，一个顶级命名空间对应多个目录路径，具体形式可以查看上面我们讲的 autoload_static 的 $prefixesPsr0 。如果没有顶级命名空间，就只存储一个路径名，以便在后面尝试加载。

#### PSR4标准

autoload_psr4

```php
<?php
    return array(
    'XdgBaseDir\\'
		=> array($vendorDir . '/dnoegel/php-xdg-base-dir/src'),

    'Webmozart\\Assert\\'
		=> array($vendorDir . '/webmozart/assert/src'),

    'TijsVerkoyen\\CssToInlineStyles\\'
		=> array($vendorDir . '/tijsverkoyen/css-to-inline-styles/src'),

    'Tests\\'
		=> array($baseDir . '/tests'),

    'Symfony\\Polyfill\\Mbstring\\'
		=> array($vendorDir . '/symfony/polyfill-mbstring'),
    ...
    )
```

PSR4标准的初始化接口:

```php
<?php
	public function setPsr4($prefix, $paths)
	{
	    if (!$prefix) {
	        $this->fallbackDirsPsr4 = (array) $paths;
	    } else {
	        $length = strlen($prefix);
	        if ('\\' !== $prefix[$length - 1]) {
	            throw new \InvalidArgumentException(
                  "A non-empty PSR-4 prefix must end with a namespace separator."
                );
	        }
	        $this->prefixLengthsPsr4[$prefix[0]][$prefix] = $length;
	        $this->prefixDirsPsr4[$prefix] = (array) $paths;
	    }
	}
```
PSR4初始化接口也很简单。如果没有顶级命名空间，就直接保存目录。如果有命名空间的话，要保证顶级命名空间最后是 `\` ，然后分别保存

	( 前缀 -> 顶级命名空间，顶级命名空间 -> 顶级命名空间长度 )
	( 顶级命名空间 -> 目录 )

这两个映射数组。具体形式可以查看上面我们讲的 `autoload_static` 的 prefixLengthsPsr4 、 $prefixDirsPsr4 。

#### 傻瓜式命名空间映射

autoload_classmap：

```php
<?php
public static $classMap = array (
    'App\\Console\\Kernel'
		=> __DIR__ . '/../..' . '/app/Console/Kernel.php',

    'App\\Exceptions\\Handler'
		=> __DIR__ . '/../..' . '/app/Exceptions/Handler.php',
    ...
)
```

addClassMap:

```php  
<?php
	public function addClassMap(array $classMap)
	{
	    if ($this->classMap) {
	        $this->classMap = array_merge($this->classMap, $classMap);
	    } else {
	        $this->classMap = $classMap;
	    }
	}
```
这个最简单，就是整个命名空间与目录之间的映射。

# 结语

其实我很想接着写下下去，但是这样会造成篇幅过长，所以我就把自动加载的注册和运行放到下一篇文章了。我们回顾一下，这篇文章主要讲了：

	1.框架如何启动composer自动加载;
	2.composer自动加载分为5部分；

其实说是5部分，真正重要的就两部分——初始化与注册。初始化负责顶层命名空间的目录映射，注册负责实现顶层以下的命名空间映射规则。


 Written with [StackEdit](https://stackedit.io/).
