# 前言

本文主要介绍 `laravel` 加载 `config` 配置文件的相关源码。

# config 配置文件的加载

config 配置文件由类 `\Illuminate\Foundation\Bootstrap\LoadConfiguration::class` 完成：

```php
class LoadConfiguration
{
	public function bootstrap(Application $app)
    {
        $items = [];

        if (file_exists($cached = $app->getCachedConfigPath())) {
            $items = require $cached;

            $loadedFromCache = true;
        }

        $app->instance('config', $config = new Repository($items));

        if (! isset($loadedFromCache)) {
            $this->loadConfigurationFiles($app, $config);
        }

        $app->detectEnvironment(function () use ($config) {
            return $config->get('app.env', 'production');
        });

        date_default_timezone_set($config->get('app.timezone', 'UTC'));

        mb_internal_encoding('UTF-8');
    }
}
```

可以看到，配置文件的加载步骤：

- 加载缓存
- 若缓存不存在，则利用函数 `loadConfigurationFiles` 加载配置文件
- 加载环境变量、时间区、编码方式

函数 `loadConfigurationFiles` 用于加载配置文件：

```php
protected function loadConfigurationFiles(Application $app, RepositoryContract $repository)
{
    foreach ($this->getConfigurationFiles($app) as $key => $path) {
        $repository->set($key, require $path);
   }
}
```

加载配置文件有两部分：搜索配置文件、加载配置文件的数组变量值

## 搜索配置文件

`getConfigurationFiles` 可以根据配置文件目录搜索所有的 `php` 为后缀的文件，并将其转化为 `files` 数组，其 `key` 为目录名以字符 `.` 为连接的字符串 ，`value` 为文件真实路径：

```php
protected function getConfigurationFiles(Application $app)
{
    $files = [];

    $configPath = realpath($app->configPath());

    foreach (Finder::create()->files()->name('*.php')->in($configPath) as $file) {
        $directory = $this->getNestedDirectory($file, $configPath);

        $files[$directory.basename($file->getRealPath(), '.php')] = $file->getRealPath();
    }

    return $files;
}

protected function getNestedDirectory(SplFileInfo $file, $configPath)
{
    $directory = $file->getPath();

    if ($nested = trim(str_replace($configPath, '', $directory), DIRECTORY_SEPARATOR)) {
        $nested = str_replace(DIRECTORY_SEPARATOR, '.', $nested).'.';
    }

    return $nested;
}
```

## 加载配置文件数组

加载配置文件由类 `Illuminate\Config\Repository\LoadConfiguration` 完成：

```php
class Repository
{
	public function set($key, $value = null)
    {
        $keys = is_array($key) ? $key : [$key => $value];

        foreach ($keys as $key => $value) {
            Arr::set($this->items, $key, $value);
        }
    }
}
```
加载配置文件时间上就是将所有配置文件的数值放入一个巨大的多维数组中，这一部分由类 `Illuminate\Support\Arr` 完成：

```php
class Arr
{
	public static function set(&$array, $key, $value)
    {
        if (is_null($key)) {
            return $array = $value;
        }

        $keys = explode('.', $key);

        while (count($keys) > 1) {
            $key = array_shift($keys);

            if (! isset($array[$key]) || ! is_array($array[$key])) {
                $array[$key] = [];
            }

            $array = &$array[$key];
        }

        $array[array_shift($keys)] = $value;

        return $array;
    }
}
```

例如 `dir1.dir2.app` ,配置文件会生成 `$array[dir1][dir2][app]` 这样的数组。

# 配置文件数值的获取

当我们利用全局函数 `config` 来获取配置值的时候：

```php
function config($key = null, $default = null)
{
    if (is_null($key)) {
        return app('config');
    }

    if (is_array($key)) {
        return app('config')->set($key);
    }

    return app('config')->get($key, $default);
}
```

配置文件的获取和加载类似，都是将字符串转为多维数组，然后获取具体数组值：

```php
public static function get($array, $key, $default = null)
{
    if (! static::accessible($array)) {
        return value($default);
    }

    if (is_null($key)) {
        return $array;
    }

    if (static::exists($array, $key)) {
        return $array[$key];
    }

    foreach (explode('.', $key) as $segment) {
        if (static::accessible($array) && static::exists($array, $segment)) {
            $array = $array[$segment];
        } else {
            return value($default);
        }
    }

    return $array;
}
```