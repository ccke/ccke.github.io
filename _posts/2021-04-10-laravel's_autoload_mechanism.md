---
title: Laravel的类自动加载机制
author: chukeey
date: 2021-04-10 16:00:00 +0800
categories: [技术, php, laravel框架]
tags: [php, laravel, autoload]
pin: false
---

### 1. 背景
​	   在PHP开发过程中，如果希望从外部引入一个class，通常会使用include和require方法，去把定义这个class的文件包含进来。

include 和 require 除了处理错误的方式不同之外，在其他方面都是相同的：

> - require 生成一个致命错误（E_COMPILE_ERROR），在错误发生后脚本会停止执行。**（推荐使用）**
> - include 生成一个警告（E_WARNING），在错误发生后脚本会继续执行。

​        这个在小规模开发的时候，没什么大问题；但在大型的开发项目中，这么做会产生大量的require或者include方法调用，不仅降低效率，而且使得代码难以维护，况且require_once的代价很大。
​        在PHP5之前，各个PHP框架如果要实现类的自动加载，一般都是按照某种约定自己实现一个遍历目录，自动加载所有符合约定规则的文件的类或函数。 当然，PHP5之前对面向对象的支持并不是太好，类的使用也没有现在频繁。 在PHP5后，当加载PHP类时，如果类所在文件没有被包含进来，或者类名出错，Zend引擎会自动调用 **\__autoload函数**。此函数需要用户自己实现 __autoload函数。 在PHP5.1.2版本后，可以使用**spl_autoload_register函数**自定义自动加载处理函数。当没有调用此函数，默认情况下会使用SPL自定义的**spl_autoload函数**。

### 2. php自动加载

#### 2.1 __autoload示例：

```php
<?php

function __autoload($className) {   
    echo '__autload class:', $className, PHP_EOL; 
} 

new Demo();

// 输出：
// __autload class:Demo
// PHP Fatal error:  Uncaught Error: Class 'Demo' not found in XXX\test.php:8
```

​		我们一般使用_autoload自动加载类如下：

```php
<?php

function __autoload($className) {   
    require_once ($className . '.class.php');
} 

new Demo();
```

　　我们可以看出_autoload至少要做三件事情：

> - 第一件事是根据类名确定类文件名
>
>
> - 第二件事是确定类文件所在的磁盘路径（在我们的例子是最简单的情况，类与调用它们的PHP程序文件在同一个文件夹下）
>
> - 第三件事是将类从磁盘文件中加载到系统中

​		第三步最简单，只需要使用include/require即可。要实现第一步，第二步的功能，必须在开发时约定类名与磁盘文件的映射方法，只有这样我们才能根据类名找到它对应的磁盘文件。 

​		因此，当有大量的类文件要包含的时候，我们只要确定相应的规则，然后在\__autoload()函数中，将类名与实际的磁盘文件对应起来，就可以实现**lazy loading**的效果。从这里我们也可以看出__autoload()函数的实现中**最重要的是类名与实际的磁盘文件映射规则的实现**。 

​        但现在问题来了，假如在一个系统的实现中，需要使用很多其它的类库，这些类库可能是由不同的开发工程师开发，其类名与实际的磁盘文件的映射规则不尽相同。这时假如要实现类库文件的自动加载，由于**\__autoload() 是全局函数只能定义一次，就必须在__autoload()函数中将所有的映射规则全部实现**，因此该函数有可能会非常复杂，甚至无法实现，即便能够实现，也会给将来的维护和系统效率带来很大的负面影响。

​		在这种情况下，在PHP5引入SPL标准库，一种新的解决方案，使用一个 \__autoload函数的队列 ，不同的映射关系写到不同的 __autoload函数中去，然后统一注册统一管理，即**spl_autoload_register()函数**。

#### 2.2 spl_autoload_register()函数

​		此函数的功能就是把函数注册至SPL的\__autoload函数队列中，并会将Zend Engine中的[__autoload()](https://www.php.net/manual/zh/function.autoload.php)函数取代为[spl_autoload()](https://www.php.net/manual/zh/function.spl-autoload.php)或[spl_autoload_call()](https://www.php.net/manual/zh/function.spl-autoload-call.php)。下面的例子可以看出：

```php
<?php

function __autoload($className) {
    echo '__autoload class:', $className, PHP_EOL;
}

function classLoader($className) {
    echo 'SPL load class:', $className, PHP_EOL;
}

spl_autoload_register('classLoader');

new Demo();

//结果：SPL load class:Demo
```

> 语法：bool spl_autoload_register ( [callback $autoload_function] )  接受两个参数：一个是添加到自动加载栈的函数，另外一个是加载器不能找到这个类时是否抛出异常的标志。
>
> 第一个参数是可选的，并且默认指向spl_autoload()函数，这个函数会自动在路径中查找具有小写类名和.php扩展或者.ini扩展名，或者任何注册到spl_autoload_extensions()函数中的其它扩展名的文件。

```php
<?php

class Test
{
    public static function loader($className)
    {
        $classFile = strtolower($className) . ".php";
        if (file_exists($classFile)){
            require_once($classFile);
        }
    }
}

// 方法为静态方法
spl_autoload_register('Test::loader');

new Demo();
```

 		一旦调用spl_autoload_register()函数，当调用未定义类时，系统会按顺序调用注册到spl_autoload_register()函数的所有函数，而不是自动调用__autoload()函数。如果要避免这种情况，需采用一种更加安全的spl_autoload_register()函数的初始化调用方法：

```php
if (false === spl_autoload_functions()) {    
    if (function_exists('__autoload')) {    
        spl_autoload_register('__autoload', false);    
    }    
 }   
```

​		**spl_autoload_functions()函数会返回已注册函数的一个数组**，如果SPL自动加载队列还没有被初始化，它会返回布尔值false。然后，检查是否有一个名为__autoload()的函数存在，如果存在，可以将它注册为自动加载队列中的第一个函数，从而保留它的功能。之后，可以继续注册自动加载函数。

​		还可以调用spl_autoload_register()函数以注册一个回调函数，而不是为函数提供一个字符串名称。如提供一个如```array('class', 'method')```这样的数组，使得可以使用某个对象的方法。

​		下一步，通过调用spl_autoload_call('className')函数，可以手动调用加载器，而不用尝试去使用那个类。这个函数可以和函数class_exists('className', false)组合在一起使用以尝试去加载一个类，并且在所有的自动加载器都不能找到那个类的情况下失败。

```php
if (spl_autoload_call('className') && class_exists('className', false)) {    
  
} else { 
    
}    
```

> **结语：**SPL自动加载功能是由spl_autoload()，spl_autoload_register()，spl_autoload_functions()，spl_autoload_extensions()和spl_autoload_call()函数提供的。

### 3. composer自动加载

#### 3.1 PSR规范

​		与php自动加载相关的规范是 PSR4，在说 PSR4 之前先介绍一下 PSR 标准。

> PSR 标准的发明和推出组织是：PHP-FIG，它的网站是：www.php-fig.org。由几位开源框架的开发者成立于 2009 年，从那开始也选取了很多其他成员进来，虽然不是 “官方” 组织，但也代表了社区中不小的一块。组织的目的在于：以最低程度的限制，来统一各个项目的编码规范，避免各家自行发展的风格阻碍了程序员开发的困扰，于是大伙发明和总结了 PSR，PSR 是 PHP Standards Recommendation 的缩写。

​		截止到目前为止，总共有 14 套 PSR 规范，其中有 7 套PSR规范已通过表决并推出使用，分别是：

- **PSR-0 自动加载标准**（已废弃，一些旧的第三方库还有在使用）
  PSR-1 基础编码标准
- PSR-2 编码风格向导

- PSR-3 日志接口

- **PSR-4 自动加载的增强版**，替换掉了 PSR-0

- PSR-6 缓存接口规范

- PSR-7 HTTP 消息接口规范


#### 3.2 PSR4 标准
​		2013 年底，PHP-FIG 推出了第 5 个规范——PSR-4。

​		PSR-4 规范了如何指定文件路径从而自动加载类定义，同时规范了自动加载文件的位置。

> 1）一个完整的类名需具有以下结构：
> \<命名空间>\<子命名空间>\<类名>
>
> - 完整的类名必须要有一个顶级命名空间，被称为 "vendor namespace"；
> - 完整的类名可以有一个或多个子命名空间；
> - 完整的类名必须有一个最终的类名；
> - 完整的类名中任意一部分中的下滑线都是没有特殊含义的；
> - 完整的类名可以由任意大小写字母组成；
> - 所有类名都必须是大小写敏感的。
>
> 2）根据完整的类名载入相应的文件
> - 完整的类名中，去掉最前面的命名空间分隔符，前面连续的一个或多个命名空间和子命名空间，作为「命名空间前缀」，其必须与至少一个「文件基目录」相对应；
>
> - 紧接命名空间前缀后的子命名空间 必须 与相应的「文件基目录」相匹配，其中的命名空间分隔符将作为目录分隔符；
>
> - 末尾的类名必须与对应的以 .php 为后缀的文件同名；
>
> - 自动加载器（autoloader）的实现一定不可抛出异常、一定不可触发任一级别的错误信息以及不应该有返回值。
>
> 3) 例子
> PSR-4风格：
>
> - 类名：SymfonyCoreRequest
> - 命名空间前缀：SymfonyCore
> - 文件基目录：./vendor/Symfony/Core/
> - 文件路径：./vendor/Symfony/Core/Request.php
>
> 目录结构：
> -vendor/
> | -vendor_name/
> | | -package_name/
> | | | -src/
> | | | | -ClassName.php       # Vendor_Name\Package_Name\ClassName
> | | | -tests/
> | | | | -ClassNameTest.php   # Vendor_Name\Package_Name\ClassNameTest

### 4. laravel自动加载



### 5. 参考目录

- [PHP的类自动加载机制]: https://blog.csdn.net/hguisu/article/details/7463333

- [深入解析 composer 的自动加载原理]: https://segmentfault.com/a/1190000014948542
