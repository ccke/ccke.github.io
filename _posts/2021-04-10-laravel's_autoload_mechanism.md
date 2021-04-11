---
title: Laravel的类自动加载机制
author: chukeey
date: 2021-04-10 16:00:00 +0800
categories: [技术, php, laravel框架]
tags: [php, laravel, autoload]
pin: false
---

### 1. 背景
在PHP开发过程中，如果希望从外部引入一个class，通常会使用include和require方法，去把定义这个class的文件包含进来。

include 和 require 除了处理错误的方式不同之外，在其他方面都是相同的：

> - require 生成一个致命错误（E_COMPILE_ERROR），在错误发生后脚本会停止执行。**（推荐使用）**
> - include 生成一个警告（E_WARNING），在错误发生后脚本会继续执行。

这个在小规模开发的时候，没什么大问题；但在大型的开发项目中，这么做会产生大量的require或者include方法调用，不仅降低效率，而且使得代码难以维护，况且require_once的代价很大。
在PHP5之前，各个PHP框架如果要实现类的自动加载，一般都是按照某种约定自己实现一个遍历目录，自动加载所有符合约定规则的文件的类或函数。 当然，PHP5之前对面向对象的支持并不是太好，类的使用也没有现在频繁。 在PHP5后，当加载PHP类时，如果类所在文件没有被包含进来，或者类名出错，Zend引擎会自动调用 **\__autoload函数**。此函数需要用户自己实现 __autoload函数。 在PHP5.1.2版本后，可以使用**spl_autoload_register函数**自定义自动加载处理函数。当没有调用此函数，默认情况下会使用SPL自定义的**spl_autoload函数**。

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

我们一般使用_autoload自动加载类如下：

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

第三步最简单，只需要使用include/require即可。要实现第一步，第二步的功能，必须在开发时约定类名与磁盘文件的映射方法，只有这样我们才能根据类名找到它对应的磁盘文件。

因此，当有大量的类文件要包含的时候，我们只要确定相应的规则，然后在\__autoload()函数中，将类名与实际的磁盘文件对应起来，就可以实现**lazy loading**的效果。从这里我们也可以看出__autoload()函数的实现中**最重要的是类名与实际的磁盘文件映射规则的实现**。

但现在问题来了，假如在一个系统的实现中，需要使用很多其它的类库，这些类库可能是由不同的开发工程师开发，其类名与实际的磁盘文件的映射规则不尽相同。这时假如要实现类库文件的自动加载，由于**\__autoload() 是全局函数只能定义一次，就必须在__autoload()函数中将所有的映射规则全部实现**，因此该函数有可能会非常复杂，甚至无法实现，即便能够实现，也会给将来的维护和系统效率带来很大的负面影响。

在这种情况下，在PHP5引入SPL标准库，一种新的解决方案，使用一个 \__autoload函数的队列 ，不同的映射关系写到不同的 __autoload函数中去，然后统一注册统一管理，即**spl_autoload_register()函数**。

#### 2.2 spl_autoload_register()函数

此函数的功能就是把函数注册至SPL的\__autoload函数队列中，并会将Zend Engine中的[__autoload()](https://www.php.net/manual/zh/function.autoload.php)函数取代为[spl_autoload()](https://www.php.net/manual/zh/function.spl-autoload.php)或[spl_autoload_call()](https://www.php.net/manual/zh/function.spl-autoload-call.php)。下面的例子可以看出：

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

**spl_autoload_functions()函数会返回已注册函数的一个数组**，如果SPL自动加载队列还没有被初始化，它会返回布尔值false。然后，检查是否有一个名为__autoload()的函数存在，如果存在，可以将它注册为自动加载队列中的第一个函数，从而保留它的功能。之后，可以继续注册自动加载函数。

还可以调用spl_autoload_register()函数以注册一个回调函数，而不是为函数提供一个字符串名称。如提供一个如```array('class', 'method')```这样的数组，使得可以使用某个对象的方法。

下一步，通过调用spl_autoload_call('className')函数，可以手动调用加载器，而不用尝试去使用那个类。这个函数可以和函数class_exists('className', false)组合在一起使用以尝试去加载一个类，并且在所有的自动加载器都不能找到那个类的情况下失败。

```php
if (spl_autoload_call('className') && class_exists('className', false)) {

} else {

}
```

> **结语：**SPL自动加载功能是由spl_autoload()，spl_autoload_register()，spl_autoload_functions()，spl_autoload_extensions()和spl_autoload_call()函数提供的。

### 3. composer自动加载

#### 3.1 PSR规范

与php自动加载相关的规范是 PSR4，在说 PSR4 之前先介绍一下 PSR 标准。

> PSR 标准的发明和推出组织是：PHP-FIG，它的网站是：www.php-fig.org。由几位开源框架的开发者成立于 2009 年，从那开始也选取了很多其他成员进来，虽然不是 “官方” 组织，但也代表了社区中不小的一块。组织的目的在于：以最低程度的限制，来统一各个项目的编码规范，避免各家自行发展的风格阻碍了程序员开发的困扰，于是大伙发明和总结了 PSR，PSR 是 PHP Standards Recommendation 的缩写。

截止到目前为止，总共有 14 套 PSR 规范，其中有 7 套PSR规范已通过表决并推出使用，分别是：

- **PSR-0 自动加载标准**（已废弃，一些旧的第三方库还有在使用）
  PSR-1 基础编码标准
- PSR-2 编码风格向导

- PSR-3 日志接口

- **PSR-4 自动加载的增强版**，替换掉了 PSR-0

- PSR-6 缓存接口规范

- PSR-7 HTTP 消息接口规范


#### 3.2 PSR4 标准
2013 年底，PHP-FIG 推出了第 5 个规范——PSR-4。

PSR-4 规范了如何指定文件路径从而自动加载类定义，同时规范了自动加载文件的位置。

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

#### 3.3 Composer依赖管理

Composer 不是一个包管理器。是的，它涉及 "packages" 和 "libraries"，但它在每个项目的基础上进行管理，在你项目的某个目录中（例如 `vendor`）进行安装。默认情况下它不会在全局安装任何东西。因此，这**仅仅是一个依赖管理**。

这种想法并不新鲜，Composer 受到了 node's npm 和 ruby's bundler 的强烈启发。而当时 PHP 下并没有类似的工具。

**Composer 将这样为你解决问题：**

a) 你有一个项目依赖于若干个库。

b) 其中一些库依赖于其他库。

c) 你声明你所依赖的东西。

d) Composer 会找出哪个版本的包需要安装，并安装它们（将它们下载到你的项目中）。

**声明依赖关系**
比方说，你正在创建一个项目，你需要一个库来做日志记录。你决定使用 monolog。为了将它添加到你的项目中，你所需要做的就是创建一个 composer.json 文件，其中描述了项目的依赖关系。

```php
{
    "require": {
        "monolog/monolog": "1.2.*"
    }
}
```

我们只要指出我们的项目需要一些 monolog/monolog 的包，从 1.2 开始的任何版本。

> 简单的说，Composer 帮我们做了这些事：
>
> - 下载好了符合 PSR0/PSR4标准 的第三方库，并把文件放在相应位置（vendor 目录下）；
>
> - 写了 __autoload() 函数，注册到了 spl_register() 函数，当我们想用第三方库的时候直接使用命名空间即可。
>
> **那么当我们想要写自己的命名空间的时候，该怎么办呢？**
>
> 1）按照 PSR4标准 命名我们的命名空间，放置我们的文件；
>
> 2）在 composer 里面写好顶级域名与具体目录的映射，就可以享用 composer 的便利了。
>
> 当然如果有一个非常棒的框架，我们会惊喜地发现，在 composer 里面写顶级域名映射这事我们也不用做了，框架已经帮我们写好了顶级域名映射了，我们只需要在框架里面新建文件，在新建的文件中写好命名空间，就可以在任何地方 use 我们的命名空间了。

#### 3.4 Composer自动加载

首先，我们先大致了解一下Composer自动加载所用到的源文件：

```php
autoload_real.php: 自动加载功能的引导类。
// 任务是composer加载类的初始化（顶级命名空间与文件路径映射初始化）和注册（spl_autoload_register()）

ClassLoader.php: composer加载类
// composer自动加载功能的核心类

autoload_static.php: 顶级命名空间初始化类
// 用于给核心类初始化顶级命名空间

autoload_classmap.php: 自动加载的最简单形式
// 有完整的命名空间和文件目录的映射

autoload_files.php: 用于加载全局函数的文件
// 存放各个全局函数所在的文件路径名

autoload_namespaces.php: 符合PSR0标准的自动加载文件
// 存放着顶级命名空间与文件的映射

autoload_psr4.php: 符合PSR4标准的自动加载文件
// 存放着顶级命名空间与文件的映射
```

##### 3.4.1 autoload_real.php 引导类

在 vendor 目录下的 autoload.php 文件中我们可以看出，程序主要调用了引导类的静态方法 getLoader() 来达到自动加载的目的，我们接着看看这个函数：

```php
/**
 * @return \Composer\Autoload\ClassLoader
 */
public static function getLoader()
{
    // 经典单例模式
    if (null !== self::$loader) {
        return self::$loader;
    }

    // 平台检查：当前composer版本需要PHP版本 >= 7.2.5
    require __DIR__ . '/platform_check.php';

    // 获得自动加载核心类对象
    spl_autoload_register(array('ComposerAutoloaderInit59119be9e78c643dbb9f5087be96e6d6', 'loadClassLoader'), true, true);
    self::$loader = $loader = new \Composer\Autoload\ClassLoader(\dirname(\dirname(__FILE__)));
    spl_autoload_unregister(array('ComposerAutoloaderInit59119be9e78c643dbb9f5087be96e6d6', 'loadClassLoader'));

    // 初始化自动加载核心类对象
    $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
    if ($useStaticLoader) {
        require __DIR__ . '/autoload_static.php';

        call_user_func(\Composer\Autoload\ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::getInitializer($loader));
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

    // 注册自动加载核心类对象
    $loader->register(true);

    // 自动加载全局函数
    if ($useStaticLoader) {
        $includeFiles = Composer\Autoload\ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$files;
    } else {
        $includeFiles = require __DIR__ . '/autoload_files.php';
    }
    foreach ($includeFiles as $fileIdentifier => $file) {
        composerRequire59119be9e78c643dbb9f5087be96e6d6($fileIdentifier, $file);
    }

    return $loader;
}
```

从上面可以看出，我们可以把自动加载引导类分为6步：

**1）经典单例模式**

```php
if (null !== self::$loader) {
	return self::$loader;
}
```

**2）平台检查**

```php
<?php

// platform_check.php @generated by Composer

$issues = array();

if (!(PHP_VERSION_ID >= 70205)) {
    $issues[] = 'Your Composer dependencies require a PHP version ">= 7.2.5". You are running ' . PHP_VERSION . '.';
}

if ($issues) {
    if (!headers_sent()) {
        header('HTTP/1.1 500 Internal Server Error');
    }
    if (!ini_get('display_errors')) {
        if (PHP_SAPI === 'cli' || PHP_SAPI === 'phpdbg') {
            fwrite(STDERR, 'Composer detected issues in your platform:' . PHP_EOL.PHP_EOL . implode(PHP_EOL, $issues) . PHP_EOL.PHP_EOL);
        } elseif (!headers_sent()) {
            echo 'Composer detected issues in your platform:' . PHP_EOL.PHP_EOL . str_replace('You are running '.PHP_VERSION.'.', '', implode(PHP_EOL, $issues)) . PHP_EOL.PHP_EOL;
        }
    }
    trigger_error(
        'Composer detected issues in your platform: ' . implode(' ', $issues),
        E_USER_ERROR
    );
}
```

**3）构造ClassLoader核心类**

```php
spl_autoload_register(array('ComposerAutoloaderInit59119be9e78c643dbb9f5087be96e6d6', 'loadClassLoader'), true, true);
    self::$loader = $loader = new \Composer\Autoload\ClassLoader(\dirname(\dirname(__FILE__)));
    spl_autoload_unregister(array('ComposerAutoloaderInit59119be9e78c643dbb9f5087be96e6d6', 'loadClassLoader')
```

loadClassLoader()函数：

```php
public static function loadClassLoader($class)
{
	if ('Composer\Autoload\ClassLoader' === $class) {
    	require __DIR__ . '/ClassLoader.php';
    }
}
```

从程序里面我们可以看出，composer 先向 PHP 自动加载机制注册了一个函数，这个函数 require 了 ClassLoader 文件。成功 new 出该文件中核心类 ClassLoader() 后，又销毁了该函数。

> **为什么不直接require，而要这么麻烦？**
>
> 原因就是怕有的用户也定义了个 \Composer\Autoload\ClassLoader 命名空间，导致自动加载错误文件。那为什么不跟引导类一样用个 hash 呢？因为这个类是可以复用的，框架允许用户使用这个类。

**4）初始化核心类对象**

```php
$useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
if ($useStaticLoader) {
    require __DIR__ . '/autoload_static.php';

    call_user_func(\Composer\Autoload\ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::getInitializer($loader));
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

> 初始化的方法有两种：
>
> - 使用autoload_static进行静态初始化；
> - 调用核心类接口初始化。

**a. 使用autoload_static进行静态初始化 ( PHP >= 5.6 ，并且不支持 HHVM 虚拟机)**

我们深入 autoload_static.php 这个文件发现这个文件定义了一个用于静态初始化的类，名字叫 ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6，仍然为了避免冲突加了 hash 值。这个类很简单：

```php
<?php

// autoload_static.php @generated by Composer

namespace Composer\Autoload;

class ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6
{
     public static $files = array(...);
     public static $prefixLengthsPsr4 = array(...);
     public static $prefixDirsPsr4 = array(...);
     public static $prefixesPsr0 = array(...);
     public static $classMap = array (...);

	public static function getInitializer(ClassLoader $loader)
    {
        return \Closure::bind(function () use ($loader) {
            $loader->prefixLengthsPsr4 = ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$prefixLengthsPsr4;
            $loader->prefixDirsPsr4 = ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$prefixDirsPsr4;
            $loader->prefixesPsr0 = ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$prefixesPsr0;
            $loader->classMap = ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$classMap;

        }, null, ClassLoader::class);
    }
}
```

这个静态初始化类的核心就是 getInitializer() 函数，它将自己类中的顶级命名空间映射给了 ClassLoader类。值得注意的是这个函数返回的是一个匿名函数，为什么呢？**原因就是 ClassLoader类 中的 prefixLengthsPsr4 、prefixDirsPsr4等等方法都是private的。**普通的函数没办法给类的 private 成员变量赋值。**利用匿名函数的绑定功能就可以将把匿名函数转为 ClassLoader类 的成员函数**。

> 关于匿名函数的绑定功能见：https://www.php.net/manual/zh/closure.bind.php

可以看到顶级命名空间初始化的关键是：$prefixLengthsPsr4、$prefixDirsPsr4、$prefixesPsr0、$classMap，接下来我们分别看下这些文件：

**PSR0 顶级命名空间映射：**

```php
<?php
	public static $prefixesPsr0 = array (
        'M' =>
        array (
            'Mockery' =>
            array (
                0 => __DIR__ . '/..' . '/mockery/mockery/library',
            ),
        ),
        'H' =>
        array (
            'Highlight\\' =>
            array (
                0 => __DIR__ . '/..' . '/scrivo/highlight.php',
            ),
            'HighlightUtilities\\' =>
            array (
                0 => __DIR__ . '/..' . '/scrivo/highlight.php',
            ),
        ),
    );
```

为了快速找到顶级命名空间，我们这里使用命名空间第一个字母作为前缀索引。这个映射的用法比较明显，假如我们有Highlight/Language 这样的命名空间，首先通过首字母H，找到如下数组：

```php
<?php
  	'H' =>
        array (
            'Highlight\\' =>
            array (
                0 => __DIR__ . '/..' . '/scrivo/highlight.php',
            ),
            'HighlightUtilities\\' =>
            array (
                0 => __DIR__ . '/..' . '/scrivo/highlight.php',
            ),
        ),
```

然后我们就会遍历这个数组来和 Highlight/Language 比较，发现第2个 HighlightUtilities 不符合，第1个 Highlight 符合，然后得到了映射目录（映射目录可能不止一个）：

```php
<?php
    array (0 => __DIR__ . '/..' . '/scrivo/highlight.php',),
```
我们会接着遍历这个数组，尝试 ```__DIR__ . '/..' . '/scrivo/highlight.php' ```是否存在，如果不存在接着遍历数组（这个例子数组只有一个元素），如果数组遍历完都没有，就会加载失败。

**PSR4标准顶级命名空间映射数组：**

```php
<?php
  public static $prefixLengthsPsr4 = array (
        'v' =>
        array (
            'voku\\' => 5,
        ),
        'p' =>
        array (
            'phpDocumentor\\Reflection\\' => 25,
        ),
  ...);

  public static $prefixDirsPsr4 = array (
        'voku\\' =>
        array (
            0 => __DIR__ . '/..' . '/voku/portable-ascii/src/voku',
        ),
        'phpDocumentor\\Reflection\\' =>
        array (
            0 => __DIR__ . '/..' . '/phpdocumentor/reflection-common/src',
            1 => __DIR__ . '/..' . '/phpdocumentor/type-resolver/src',
            2 => __DIR__ . '/..' . '/phpdocumentor/reflection-docblock/src',
        ),
  ...);
```

PSR4 标准顶级命名空间映射用了两个数组，第一个和 PSR0 一样用命名空间第一个字母作为前缀索引，然后是 顶级命名空间，但是最终并不是文件路径，而是 顶级命名空间的长度。为什么呢？**因为 PSR4 标准是用顶级命名空间目录替换顶级命名空间，所以获得顶级命名空间的长度很重要**。

PSR0中顶级命名空间目录直接加到命名空间前面就可以得到路径：

```php
Highlight/Language => __DIR__ . '/..' . '/scrivo/highlight.php/Highlight/Language.php'
```

而 PSR4标准却是用顶级命名空间目录替换顶级命名空间，得到路径：

```php
phpDocumentor/Reflection/Element => __DIR__ . '/..' . '/phpdocumentor/reflection-common/src/Element.php'
```

具体的用法：假如我们找 phpDocumentor/Reflection/Element 这个命名空间，和 PSR0 一样通过前缀索引和字符串匹配我们得到了

```php
<?php
	'p' =>
        array (
            'phpDocumentor\\Reflection\\' => 25,
        ),
```

这条记录，键是顶级命名空间，值是命名空间的长度。拿到顶级命名空间后去 $prefixDirsPsr4数组 获取它的映射目录数组：

```php
<?php
	'phpDocumentor\\Reflection\\' =>
        array (
            0 => __DIR__ . '/..' . '/phpdocumentor/reflection-common/src',
            1 => __DIR__ . '/..' . '/phpdocumentor/type-resolver/src',
            2 => __DIR__ . '/..' . '/phpdocumentor/reflection-docblock/src',
        ),
```

然后我们就可以将命名空间 phpDocumentor/Reflection/Element 前26个字符替换成目录 ```__DIR__ . '/..' . '/phpdocumentor/reflection-common/src'``` ，我们就得到了```__DIR__ . '/..' . '/phpdocumentor/reflection-common/src/Element.php'```，先验证磁盘上这个文件是否存在，如果不存在接着遍历。如果遍历后没有找到，则加载失败。

**最简单的 classMap:**

```php
<?php
  public static $classMap = array (
        'App\\Console\\Kernel' => __DIR__ . '/../..' . '/app/Console/Kernel.php',
        'App\\Exceptions\\Handler' => __DIR__ . '/../..' . '/app/Exceptions/Handler.php',
        'App\\Http\\Controllers\\Controller' => __DIR__ . '/../..' . '/app/Http/Controllers/Controller.php',
  ...);
```

直接命名空间全名与目录的映射，没有顶级命名空间，Laravel框架的app目录下的代码命名空间映射就放在这里。

**b. 调用核心类接口初始化**

如果PHP版本低于5.6或者使用 HHVM 虚拟机环境，那么就要使用核心类的接口进行初始化。

```php
<?php
    // PSR0标准
    $map = require __DIR__ . '/autoload_namespaces.php';
	foreach ($map as $namespace => $path) {
    	$loader->set($namespace, $path);
	}

    // PSR4标准
	$map = require __DIR__ . '/autoload_psr4.php';
	foreach ($map as $namespace => $path) {
    	$loader->setPsr4($namespace, $path);
	}

    // 直接映射
	$classMap = require __DIR__ . '/autoload_classmap.php';
	if ($classMap) {
    	$loader->addClassMap($classMap);
	}
```

**PSR0标准**

autoload_namespaces.php

```php
<?php

// autoload_namespaces.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'Mockery' => array($vendorDir . '/mockery/mockery/library'),
    'Highlight\\' => array($vendorDir . '/scrivo/highlight.php'),
    'HighlightUtilities\\' => array($vendorDir . '/scrivo/highlight.php'),
);
```

PSR0标准的初始化接口：
```php
<?php

	/**
     * Registers a set of PSR-0 directories for a given prefix,
     * replacing any others previously set for this prefix.
     *
     * @param string       $prefix The prefix
     * @param array|string $paths  The PSR-0 base directories
     */
    public function set($prefix, $paths)
    {
        if (!$prefix) {
            $this->fallbackDirsPsr0 = (array) $paths;
        } else {
            $this->prefixesPsr0[$prefix[0]][$prefix] = (array) $paths;
        }
    }
```

很简单，转换后的结构同 autoload_static.php 的 $prefixesPsr0 。如果没有顶级命名空间，就只存储一个路径名，以便在后面尝试加载。

**PSR4标准**

autoload_psr4.php

```php
<?php

// autoload_psr4.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'voku\\' => array($vendorDir . '/voku/portable-ascii/src/voku'),
    'phpDocumentor\\Reflection\\' => array($vendorDir . '/phpdocumentor/reflection-common/src', $vendorDir . '/phpdocumentor/type-resolver/src', $vendorDir . '/phpdocumentor/reflection-docblock/src'),
	...);
```

PSR4标准的初始化接口：
```php
<?php

	/**
     * Registers a set of PSR-4 directories for a given namespace,
     * replacing any others previously set for this namespace.
     *
     * @param string       $prefix The prefix/namespace, with trailing '\\'
     * @param array|string $paths  The PSR-4 base directories
     *
     * @throws \InvalidArgumentException
     */
    public function setPsr4($prefix, $paths)
    {
        if (!$prefix) {
            $this->fallbackDirsPsr4 = (array) $paths;
        } else {
            $length = strlen($prefix);
            if ('\\' !== $prefix[$length - 1]) {
                throw new \InvalidArgumentException("A non-empty PSR-4 prefix must end with a namespace separator.");
            }
            $this->prefixLengthsPsr4[$prefix[0]][$prefix] = $length;
            $this->prefixDirsPsr4[$prefix] = (array) $paths;
        }
    }
```

PSR4初始化接口也很简单，转换后的结构同 autoload_static.php 的 $prefixLengthsPsr4 和 $prefixDirsPsr4 。如果没有顶级命名空间，就只存储一个路径名，以便在后面尝试加载。

**命名空间直接映射**
autoload_classmap.php

```php
<?php

// autoload_classmap.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'App\\Console\\Kernel' => $baseDir . '/app/Console/Kernel.php',
    'App\\Exceptions\\Handler' => $baseDir . '/app/Exceptions/Handler.php',
    'App\\Http\\Controllers\\Controller' => $baseDir . '/app/Http/Controllers/Controller.php',
	...);
```

addClassMap：
```php
<?php

	/**
     * @param array $classMap Class to filename map
     */
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

**5）注册自动加载核心类对象**

讲完了 Composer 自动加载功能的启动与初始化，经过启动与初始化，自动加载核心类对象已经获得了顶级命名空间与相应目录的映射，也就是说，如果有命名空间 'App\Console\Kernel'，我们已经可以找到它对应的类文件所在位置。那么，它是什么时候被触发去找的呢？这就是 composer 自动加载的核心了。

```php
<?php

$loader->register(true);
```
\Composer\Autoload\ClassLoader::register具体实现：

```php
<?php

     /**
     * Registers this instance as an autoloader.
     *
     * @param bool $prepend Whether to prepend the autoloader or not
     */
    public function register($prepend = false)
    {
    	// 注册自动加载函数
        spl_autoload_register(array($this, 'loadClass'), true, $prepend);

        if (null === $this->vendorDir) {
            //no-op
        } elseif ($prepend) {
            self::$registeredLoaders = array($this->vendorDir => $this) + self::$registeredLoaders;
        } else {
            unset(self::$registeredLoaders[$this->vendorDir]);
            self::$registeredLoaders[$this->vendorDir] = $this;
        }
    }
```

因$prepend为true，spl_autoload_register() 会添加自动加载核心类ClassLoader的loadClass()函数到队列之首，而不是队列尾部，该函数会最先执行。这个函数负责按照PSR标准将顶层命名空间以下的内容转为对应的目录，核心类ClassLoader将loadClass()函数注册到PHP SPL中的spl_autoload_register()里面去。这样，每当PHP遇到一个不认识的命名空间的时候，PHP会自动调用注册到spl_autoload_register里面的函数队列，运行其中的每个函数，直到找到命名空间对应的文件。

**6）自动加载全局函数**

Composer不止可以自动加载命名空间，还可以加载全局函数。怎么实现的呢？很简单，把全局函数写到特定的文件里面去，在程序运行前挨个require就行了。
```php
<?php

if ($useStaticLoader) {
    $includeFiles = Composer\Autoload\ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$files;
} else {
    $includeFiles = require __DIR__ . '/autoload_files.php';
}
foreach ($includeFiles as $fileIdentifier => $file) {
    composerRequire59119be9e78c643dbb9f5087be96e6d6($fileIdentifier, $file);
}
```

跟核心类的初始化一样，全局函数自动加载也分为两种：静态初始化和普通初始化，静态加载只支持PHP5.6以上并且不支持HHVM。

**a. 静态初始化 ( PHP >= 5.6 ，并且不支持 HHVM 虚拟机)**
	     \Composer\Autoload\ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6::$files
```php
<?php

// autoload_static.php @generated by Composer

namespace Composer\Autoload;

class ComposerStaticInit59119be9e78c643dbb9f5087be96e6d6
{
    public static $files = array (
        'a4a119a56e50fbb293281d9a48007e0e' => __DIR__ . '/..' . '/symfony/polyfill-php80/bootstrap.php',
        '6e3fae29631ef280660b3cdad06f25a8' => __DIR__ . '/..' . '/symfony/deprecation-contracts/function.php',
        '0e6d7bf4a5811bfa5cf40c5ccd6fae6a' => __DIR__ . '/..' . '/symfony/polyfill-mbstring/bootstrap.php',
    ...);
```

**b. 普通初始化**
autoload_files.php

```php
<?php

// autoload_files.php @generated by Composer

$vendorDir = dirname(dirname(__FILE__));
$baseDir = dirname($vendorDir);

return array(
    'a4a119a56e50fbb293281d9a48007e0e' => $vendorDir . '/symfony/polyfill-php80/bootstrap.php',
    '6e3fae29631ef280660b3cdad06f25a8' => $vendorDir . '/symfony/deprecation-contracts/function.php',
    '0e6d7bf4a5811bfa5cf40c5ccd6fae6a' => $vendorDir . '/symfony/polyfill-mbstring/bootstrap.php',
    ...);
```

同静态初始化。

**c. 加载全局函数**

```php
<?php

class ComposerAutoloaderInit59119be9e78c643dbb9f5087be96e6d6 {
  public static function getLoader(){
      ...
      foreach ($includeFiles as $fileIdentifier => $file) {
        composerRequire59119be9e78c643dbb9f5087be96e6d6($fileIdentifier, $file);
      }
      ...
  }
}

function composerRequire59119be9e78c643dbb9f5087be96e6d6($fileIdentifier, $file)
{
    if (empty($GLOBALS['__composer_autoload_files'][$fileIdentifier])) {
        require $file;

        $GLOBALS['__composer_autoload_files'][$fileIdentifier] = true;
    }
}
```

> 这一段很有讲究，可以发掘出两个问题：
>
> **1）第一个问题：**为什么自动加载引导类的getLoader()函数不直接require $includeFiles里面的每个文件名，而要用类外面的函数composerRequire59119be9e78c643dbb9f5087be96e6d6？（顺便说下这个函数名hash仍然为了避免和用户定义函数冲突）
>
> **因为怕有人在全局函数所在的文件写$this或者self。**假如$includeFiles有个app/helper.php文件，这个helper.php文件的函数外有一行代码：$this->foo()，如果引导类在getLoader()函数直接require($file)，那么引导类就会运行这句代码，调用自己的foo()函数，这显然是错的。事实上helper.php就不应该出现$this或self这样的代码，这样写一般都是用户写错了的，一旦这样的事情发生：
>
> - 第一种情况：引导类恰好有foo()函数，那么就会莫名其妙执行了引导类的foo();
> - 第二种情况：引导类没有foo()函数，但是却甩出来引导类没有foo()方法这样的错误提示，用户不知道自己哪里错了。
>
> 把require语句放到引导类的外面，遇到$this或者self，程序就会告诉用户根本没有类，$this或self无效，错误信息更加明朗。
>
> **2）第二个问题：**为什么要用hash作为$fileIdentifier，上面的代码明显可以看出来这个变量是用来控制全局函数只被require一次的，那为什么不用require_once呢？
>
> **事实上require_once比require效率低很多，使用全局变量$GLOBALS这样控制加载会更快**。

**7）自动加载的运行**

\Composer\Autoload\ClassLoader::loadClass的具体实现：

```php
<?php

     /**
     * Loads the given class or interface.
     *
     * @param  string    $class The name of the class
     * @return bool|null True if loaded, null otherwise
     */
    public function loadClass($class)
    {
        if ($file = $this->findFile($class)) {
            includeFile($file);

            return true;
        }
    }

	/**
     * Finds the path to the file where the class is defined.
     *
     * @param string $class The name of the class
     *
     * @return string|false The path if found, false otherwise
     */
    public function findFile($class)
    {
        // class map lookup
        if (isset($this->classMap[$class])) {
            return $this->classMap[$class];
        }
        if ($this->classMapAuthoritative || isset($this->missingClasses[$class])) {
            return false;
        }
        if (null !== $this->apcuPrefix) {
            $file = apcu_fetch($this->apcuPrefix.$class, $hit);
            if ($hit) {
                return $file;
            }
        }

        $file = $this->findFileWithExtension($class, '.php');

        // Search for Hack files if we are running on HHVM
        if (false === $file && defined('HHVM_VERSION')) {
            $file = $this->findFileWithExtension($class, '.hh');
        }

        if (null !== $this->apcuPrefix) {
            apcu_add($this->apcuPrefix.$class, $file);
        }

        if (false === $file) {
            // Remember that this class does not exist.
            $this->missingClasses[$class] = true;
        }

        return $file;
    }
```

loadClass()函数主要调用findFile()函数加载文件路径，findFile()在解析命名空间的时候主要分为两部分：classMap和findFileWithExtension()函数。classMap很简单，直接看命名空间是否在映射数组中即可；麻烦的是findFileWithExtension()函数，这个函数包含了PSR0和PSR4标准的实现。

> 说明：
>
> - 查找路径成功后执行的includeFile()函数仍然是类外面的函数，并不是ClassLoader的成员函数，原理跟上面一样，防止有用户写$this或self；
> - 如果命名空间是以\开头的，要去掉\然后再匹配。

\Composer\Autoload\ClassLoader::findFileWithExtension的具体实现：
```php
<?php

    private function findFileWithExtension($class, $ext)
    {
        // PSR-4 lookup
        $logicalPathPsr4 = strtr($class, '\\', DIRECTORY_SEPARATOR) . $ext;

        $first = $class[0];
        if (isset($this->prefixLengthsPsr4[$first])) {
            $subPath = $class;
            while (false !== $lastPos = strrpos($subPath, '\\')) {
                $subPath = substr($subPath, 0, $lastPos);
                $search = $subPath . '\\';
                if (isset($this->prefixDirsPsr4[$search])) {
                    $pathEnd = DIRECTORY_SEPARATOR . substr($logicalPathPsr4, $lastPos + 1);
                    foreach ($this->prefixDirsPsr4[$search] as $dir) {
                        if (file_exists($file = $dir . $pathEnd)) {
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

        return false;
    }
```

### 4. Laravel自动加载

除了库的下载，Composer 还准备了一个自动加载文件，它可以加载 Composer 下载的库中所有的类文件。使用它，你只需要将下面这行代码添加到你项目的引导文件中：

```php
require 'vendor/autoload.php';
```

Laravel框架在其入口文件 public/index.php 中引入了composer的自动加载文件，从而达到Laravel的自动加载：

```php
<?php

/**
 * Laravel - A PHP Framework For Web Artisans
 *
 * @package  Laravel
 * @author   Taylor Otwell <taylor@laravel.com>
 */

define('LARAVEL_START', microtime(true));

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader for
| our application. We just need to utilize it! We'll simply require it
| into the script here so that we don't have to worry about manual
| loading any of our classes later on. It feels great to relax.
|
*/

// 引入composer的自动加载文件
require __DIR__.'/../vendor/autoload.php';

// 省略...
```

### 5. 参考目录

- PHP的类自动加载机制：https://blog.csdn.net/hguisu/article/details/7463333

- 深入解析 composer 的自动加载原理：https://segmentfault.com/a/1190000014948542

- composer官方文档： https://docs.phpcomposer.com/00-intro.html
