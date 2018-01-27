<p align=""><a href="https://laravel.com" target="_blank"><img src="https://laravel.com/assets/img/components/logo-laravel.svg"></a></p>


# laravel 源码详解

`laravel` 是一个非常简洁、优雅的 `PHP` 开发框架。`laravel` 中除了提供最为中心的 `Ioc` 容器之外，还提供了强大的 `路由`、`数据库模型` 等常用功能模块。

对于开发者来说，在使用 `laravel` 框架进行 `web` 开发的同时，一定很好奇 `laravel` 内部各个模块的原理，知其然更知其所以然，有助于提供开发的稳定与效率。

本项目针对 `laravel 5.4` 各个重要模块的源码进行了较为详尽的分析，希望给想要了解 `laravel` 底层原理与源码的同学一些指引。

由于本人能力有限，文章中可能会有一些问题，尽请提出意见与建议，谢谢。

本项目阅读的推荐顺序：

PHP Composer——自动加载原理

PHP Composer—— 初始化源码分析

PHP Composer-——-注册与运行源码分析

Laravel Facade——Facade 门面源码分析

Laravel Container——IoC 服务容器

Laravel Container——IoC 服务容器源码解析(服务器绑定)

Laravel Container——IoC 服务容器源码解析(服务器解析)

Laravel Container——服务容器的细节特性

Laravel HTTP——路由

Laravel HTTP——路由加载源码分析

Laravel HTTP——Pipeline中间件处理源码分析

Laravel HTTP——路由的正则编译

Laravel HTTP——路由的匹配与参数绑定

Laravel HTTP——路由中间件源码分析

Laravel HTTP——SubstituteBindings 参数绑定中间件的使用与源码解析

Laravel HTTP——控制器方法的参数构建与运行

Laravel HTTP—— RESTFul 风格路由的使用与源码分析

Laravel HTTP——重定向的使用与源码分析

Laravel ENV—— 环境变量的加载与源码解析

Laravel Config—— 配置文件的加载与源码解析

Laravel Exceptions——异常与错误处理

Laravel Providers——服务提供者的注册与启动源码解析

Laravel Database——数据库服务的启动与连接

Laravel Database——数据库的 CRUD 操作

Laravel Database——查询构造器与语法编译器源码分析(上)

Laravel Database——查询构造器与语法编译器源码分析(中)

Laravel Database——查询构造器与语法编译器源码分析(下)

Laravel Database——分页原理与源码分析

Laravel Database——Eloquent Model 源码分析(上)

Laravel Database——Eloquent Model 源码分析（下）

Laravel Database——Eloquent Model 关联源码分析

Laravel Database——Eloquent Model 模型关系加载与查询

Laravel Database——Eloquent Model 更新关联模型

Laravel Session——session 的启动与运行源码分析

Laravel Event——事件系统的启动与运行源码分析

Laravel Queue——消息队列任务与分发源码剖析

Laravel Queue——消息队列任务处理器源码剖析

Laravel Broadcast——广播系统源码剖析

Laravel Passport——OAuth2 API 认证系统源码解析

Laravel Passport——OAuth2 API 认证系统源码解析（下）












