title: Composer的Autoload源码实现——注册与运行
tags:
  - php
  - composer
  - autoload
  - 源码
categories:
  - composer
  - php
  - laravel
author: leoyang
date: 2017-03-18 12:20:00
---
# 前言
---
 &emsp;&emsp;[上一篇](http://leoyang90.cn/2017/03/13/Composer%20Autoload%20Source%20Reading%20%E2%80%94%E2%80%94%20Start%20and%20Initialize/)文章我们讲到了Composer自动加载功能的启动与初始化，经过启动与初始化，自动加载核心类对象已经获得了顶级命名空间与相应目录的映射，换句话说，如果有命名空间'App\\Console\\Kernel，我们已经知道了App\\对应的目录，接下来我们就要解决下面的就是\\Console\\Kernel这一段。
 
 #  Composer自动加载源码分析——注册
 ---
  &emsp;&emsp;我们先回顾一下自动加载引导类：
	
	 ```php
     public static function getLoader()
     {
        /***************************经典单例模式********************/
        if (null !== self::$loader) {
            return self::$loader;
        }
        
        /***********************获得自动加载核心类对象********************/
        spl_autoload_register(array('ComposerAutoloaderInit
        832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader'), true, true);
        
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        
        spl_autoload_unregister(array('ComposerAutoloaderInit
        832ea71bfb9a4128da8660baedaac82e', 'loadClassLoader'));

        /***********************初始化自动加载核心类对象********************/
        $useStaticLoader = PHP_VERSION_ID >= 50600 && 
        !defined('HHVM_VERSION');
        
        if ($useStaticLoader) {
            require_once __DIR__ . '/autoload_static.php';

            call_user_func(\Composer\Autoload\ComposerStaticInit
            832ea71bfb9a4128da8660baedaac82e::getInitializer($loader));
      
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
            $includeFiles = Composer\Autoload\ComposerStaticInit
            832ea71bfb9a4128da8660baedaac82e::$files;
        } else {
            $includeFiles = require __DIR__ . '/autoload_files.php';
        }
        
        foreach ($includeFiles as $fileIdentifier => $file) {
            composerRequire
            832ea71bfb9a4128da8660baedaac82e($fileIdentifier, $file);
        }

        return $loader;
    } 
      ```
现在我们开始引导类的第四部分：注册自动加载核心类对象。我们来看看核心类的register()函数：
	
	```php
	public function register($prepend = false)
    {
        spl_autoload_register(array($this, 'loadClass'), true, $prepend);
    }
    ```
简单到爆炸啊！一行代码实现自动加载有木有！其实奥秘都在自动加载核心类ClassLoader的loadClass()函数上，这个函数负责按照PSR标准将顶层命名空间以下的内容转为对应的目录，也就是上面所说的将'App\\Console\\Kernel中'Console\\Kernel这一段转为目录，至于怎么转的我们在下面“Composer自动加载源码分析——运行”讲。核心类ClassLoader将loadClass()函数注册到PHP SPL中的spl_autoload_register()里面去，这个函数的来龙去脉我们之前[文章](http://leoyang90.cn/2017/03/11/PHP-Composer-autoload/)讲过。这样，每当PHP遇到一个不认识的命名空间的时候，PHP会自动调用注册到spl_autoload_register里面的函数堆栈，运行其中的每个函数，直到找到命名空间对应的文件。

 #  Composer自动加载源码分析——全局函数的自动加载
 &emsp;&emsp;Composer不止可以自动加载命名空间，还可以加载全局函数。怎么实现的呢？很简单，把全局函数写到特定的文件里面去，在程序运行前挨个require就行了。这个就是composer自动加载的第五步，加载全局函数。
	 
	  ```php
	  if ($useStaticLoader) {
            $includeFiles = Composer\Autoload\ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$files;
        } else {
            $includeFiles = require __DIR__ . '/autoload_files.php';
        }
        foreach ($includeFiles as $fileIdentifier => $file) {
            composerRequire832ea71bfb9a4128da8660baedaac82e($fileIdentifier, $file);
        }
        ```
  跟核心类的初始化一样，全局函数自动加载也分为两种：静态初始化和普通初始化，静态加载只支持PHP5.6以上并且不支持HHVM。
  ## 静态初始化：
  ComposerStaticInit832ea71bfb9a4128da8660baedaac82e::$files：
	  
	   ```php
	   public static $files = array (
        '0e6d7bf4a5811bfa5cf40c5ccd6fae6a' => __DIR__ . '/..' . '/symfony/polyfill-mbstring/bootstrap.php',
        '667aeda72477189d0494fecd327c3641' => __DIR__ . '/..' . '/symfony/var-dumper/Resources/functions/dump.php',
        ...
    );
    ```
  看到这里我们可能又要有疑问了，为什么不直接放文件路径名，还要一个hash干什么呢？这个我们一会儿讲，我们这里先了解一下这个数组的结构。
 
  ##  普通初始化
  
  autoload_files:

	```php
	$vendorDir = dirname(dirname(__FILE__));
	$baseDir = dirname($vendorDir);
	
	return array(
    '0e6d7bf4a5811bfa5cf40c5ccd6fae6a' => $vendorDir . '/symfony/polyfill-mbstring/bootstrap.php',
    '667aeda72477189d0494fecd327c3641' => $vendorDir . '/symfony/var-dumper/Resources/functions/dump.php',
	   ....
	);
	```
其实跟静态初始化区别不大。

## 加载全局函数

	```php
    class ComposerAutoloaderInit832ea71bfb9a4128da8660baedaac82e{
	  public static function getLoader(){
		  ...
		  foreach ($includeFiles as $fileIdentifier => $file) {
            composerRequire832ea71bfb9a4128da8660baedaac82e($fileIdentifier, $file);
	      }
	      ...
	  }
    }

	function composerRequire832ea71bfb9a4128da8660baedaac82e($fileIdentifier, $file)
	 {
	    if (empty(\$GLOBALS['__composer_autoload_files'][\$fileIdentifier])) {
	        require $file;

	        $GLOBALS['__composer_autoload_files'][$fileIdentifier] = true;
	    }
	}
    ```

 &emsp;&emsp;这一段很有讲究，**第一个问题**：为什么自动加载引导类的getLoader()函数不直接require \$includeFiles里面的每个文件名，而要用类外面的函数composerRequire832ea71bfb9a4128da8660baedaac82e0？(顺便说下这个函数名hash仍然为了避免和用户定义函数冲突)因为怕有人在全局函数所在的文件**写\$this或者self**。
 &emsp;&emsp;假如\$includeFiles有个app/helper.php文件，这个helper.php文件的函数外有一行代码：\$this->foo()，如果引导类在getLoader()函数直接require(\$file)，那么引导类就会运行这句代码，调用自己的foo()函数，这显然是错的。事实上helper.php就不应该出现\$this或self这样的代码，这样写一般都是用户写错了的，一旦这样的事情发生，第一种情况：引导类恰好有foo()函数，那么就会莫名其妙执行了引导类的foo();第二种情况：引导类没有foo()函数，但是却甩出来引导类没有foo()方法这样的错误提示，用户不知道自己哪里错了。把require语句放到引导类的外面，遇到$this或者self，程序就会告诉用户根本没有类，$this或self无效，错误信息更加明朗。
&emsp;&emsp;**第二个问题**，为什么要用hash作为\$fileIdentifier，上面的代码明显可以看出来这个变量是用来控制全局函数只被require一次的，那为什么不用require_once呢？事实上require_once比require效率低很多，使用全局变量\$GLOBALS这样控制加载会更快。还有一个原因我猜测应该是require_once对相对路径的支持并不理想，所以composer尽量少用require_once。

 #  Composer自动加载源码分析——运行
  &emsp;&emsp;我们终于来到了核心的核心——composer自动加载的真相，命名空间如何通过composer转为对应目录文件的奥秘就在这一章。
 &emsp;&emsp;前面说过，ClassLoader的register()函数将loadClass()函数注册到PHP的SPL函数堆栈中，每当PHP遇到不认识的命名空间时就会调用函数堆栈的每个函数，直到加载命名空间成功。所以loadClass()函数就是自动加载的关键
 了。
 loadClass():

	```php
	public function loadClass($class)
    {
        if ($file = $this->findFile($class)) {
            includeFile($file);

            return true;
        }
    }

    public function findFile($class)
    {
        // work around for PHP 5.3.0 - 5.3.2 https://bugs.php.net/50731
        if ('\\' == $class[0]) {
            $class = substr($class, 1);
        }

        // class map lookup
        if (isset($this->classMap[$class])) {
            return $this->classMap[$class];
        }
        if ($this->classMapAuthoritative) {
            return false;
        }

        $file = $this->findFileWithExtension($class, '.php');

        // Search for Hack files if we are running on HHVM
        if ($file === null && defined('HHVM_VERSION')) {
            $file = $this->findFileWithExtension($class, '.hh');
        }

        if ($file === null) {
            // Remember that this class does not exist.
            return $this->classMap[$class] = false;
        }

        return $file;
    }
    ```
 &emsp;&emsp;我们看到loadClass()，主要调用findFile()函数。findFile()在解析命名空间的时候主要分为两部分：classMap和findFileWithExtension()函数。classMap很简单，直接看命名空间是否在映射数组中即可。麻烦的是findFileWithExtension()函数，这个函数包含了PSR0和PSR4标准的实现。还有个值得我们注意的是查找路径成功后includeFile()仍然类外面的函数，并不是ClassLoader的成员函数，原理跟上面一样，防止有用户写$this或self。还有就是如果命名空间是以\\开头的，要去掉\\然后再匹配。
 findFileWithExtension：
	 
	     ```php
	     private function findFileWithExtension($class, $ext)
    {
        // PSR-4 lookup
        $logicalPathPsr4 = strtr($class, '\\', DIRECTORY_SEPARATOR) . $ext;

        $first = $class[0];
        if (isset($this->prefixLengthsPsr4[$first])) {
            foreach ($this->prefixLengthsPsr4[$first] as $prefix => $length) {
                if (0 === strpos($class, $prefix)) {
                    foreach ($this->prefixDirsPsr4[$prefix] as $dir) {
                        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . substr($logicalPathPsr4, $length))) {
                            return $file;
                        }
                    }
                }
            }
        }

        // PSR-4 fallback dirs
        foreach ($this->fallbackDirsPsr4 as $dir) {
            if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr4)) {
                return $file;
            }
        }

        // PSR-0 lookup
        if (false !== $pos = strrpos($class, '\\')) {
            // namespaced class name
            $logicalPathPsr0 = substr($logicalPathPsr4, 0, $pos + 1)
                . strtr(substr($logicalPathPsr4, $pos + 1), '_', DIRECTORY_SEPARATOR);
        } else {
            // PEAR-like class name
            $logicalPathPsr0 = strtr($class, '_', DIRECTORY_SEPARATOR) . $ext;
        }

        if (isset($this->prefixesPsr0[$first])) {
            foreach ($this->prefixesPsr0[$first] as $prefix => $dirs) {
                if (0 === strpos($class, $prefix)) {
                    foreach ($dirs as $dir) {
                        if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
                            return $file;
                        }
                    }
                }
            }
        }

        // PSR-0 fallback dirs
        foreach ($this->fallbackDirsPsr0 as $dir) {
            if (file_exists($file = $dir . DIRECTORY_SEPARATOR . $logicalPathPsr0)) {
                return $file;
            }
        }

        // PSR-0 include paths.
        if ($this->useIncludePath && $file = stream_resolve_include_path($logicalPathPsr0)) {
            return $file;
        }
    }
    ```
&emsp;&emsp;下面我们通过举例来说下上面代码的流程：
&emsp;&emsp;如果我们在代码中写下'phpDocumentor\\Reflection\\example'，PHP会通过SPL调用loadClass->findFile->findFileWithExtension。首先默认用php作为文件后缀名调用findFileWithExtension函数里，利用PSR4标准尝试解析目录文件，如果文件不存在则继续用PSR0标准解析，如果解析出来的目录文件仍然不存在，但是环境是HHVM虚拟机，继续用后缀名为hh再次调用findFileWithExtension函数，如果不存在，说明此命名空间无法加载，放到classMap中设为false，以便以后更快地加载。
&emsp;&emsp;对于phpDocumentor\\Reflection\\example，当尝试利用PSR4标准映射目录时，步骤如下：

## PSR4标准加载
> - 将\\转为文件分隔符/，加上后缀php或hh，得到\$logicalPathPsr4即phpDocumentor//Reflection//example.php(hh);
> - 利用命名空间第一个字母p作为前缀索引搜索prefixLengthsPsr4数组，查到下面这个数组：
	
		```php
		p' => 
	        array (
	            'phpDocumentor\\Reflection\\' => 25,
	            'phpDocumentor\\Fake\\' => 19,
	      )
	    ```

> - 遍历这个数组，得到两个顶层命名空间phpDocumentor\\Reflection\\和phpDocumentor\\Fake\\
> - 用这两个顶层命名空间与phpDocumentor\\Reflection\\example_e相比较，可以得到phpDocumentor\\Reflection\\这个顶层命名空间
> - 在prefixLengthsPsr4映射数组中得到phpDocumentor\\Reflection\\长度为25。
> - 在prefixDirsPsr4映射数组中得到phpDocumentor\\Reflection\\的目录映射为：

	```php
	'phpDocumentor\\Reflection\\' => 
        array (
            0 => __DIR__ . '/..' . '/phpdocumentor/reflection-common/src',
            1 => __DIR__ . '/..' . '/phpdocumentor/type-resolver/src',
            2 => __DIR__ . '/..' . '/phpdocumentor/reflection-docblock/src',
        ),
    ```
> - 遍历这个映射数组，得到三个目录映射；
> - 查看 “目录+文件分隔符//+substr(\$logicalPathPsr4, \$length)”文件是否存在，存在即返回。这里就是'\__DIR\__/../phpdocumentor/reflection-common/src + /+ substr(phpDocumentor/Reflection/example_e.php(hh),25)'
> - 如果失败，则利用fallbackDirsPsr4数组里面的目录继续判断是否存在文件，具体方法是“目录+文件分隔符//+\$logicalPathPsr4”

## PSR0标准加载
如果PSR4标准加载失败，则要进行PSR0标准加载：
> - 找到phpDocumentor\\Reflection\\example_e最后“\\”的位置，将其后面文件名中’‘_’‘字符转为文件分隔符“/”,得到$logicalPathPsr0即phpDocumentor/Reflection/example/e.php(hh)
>  利用命名空间第一个字母p作为前缀索引搜索prefixLengthsPsr4数组，查到下面这个数组：

	```php
    'P' => 
        array (
            'Prophecy\\' => 
            array (
                0 => __DIR__ . '/..' . '/phpspec/prophecy/src',
            ),
            'phpDocumentor' => 
            array (
                0 => __DIR__ . '/..' . '/erusev/parsedown',
            ),
        ),
     ```
        
> - 遍历这个数组，得到两个顶层命名空间phpDocumentor和Prophecy
> - 用这两个顶层命名空间与phpDocumentor\\Reflection\\example_e相比较，可以得到phpDocumentor这个顶层命名空间
> - 在映射数组中得到phpDocumentor目录映射为'\__DIR\__ . '/..' . '/erusev/parsedown'
>- 查看 “目录+文件分隔符//+\$logicalPathPsr0”文件是否存在，存在即返回。这里就是
> “\__DIR\__ . '/..' . '/erusev/parsedown + //+ phpDocumentor//Reflection//example/e.php(hh)”
> - 如果失败，则利用fallbackDirsPsr0数组里面的目录继续判断是否存在文件，具体方法是“目录+文件分隔符//+\$logicalPathPsr0”
> - 如果仍然找不到，则利用stream_resolve_include_path()，在当前include目录寻找该文件，如果找到返回绝对路径。

# 结语
&emsp;&emsp;经过三篇文章，终于写完了PHP Composer自动加载的原理与实现，结下来我们开始讲解laravel框架下的门面Facade,这个门面功能和自动加载有着一些联系.