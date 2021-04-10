---
title: Laravel 的自动加载机制
author: chukeey
date: 2021-04-10 16:00:00 +0800
categories: [技术, 数据结构与算法]
tags: [算法, 图]
pin: false
---

### 1. 背景
在PHP开发过程中，如果希望从外部引入一个class，通常会使用include和require方法，去把定义这个class的文件包含进来。include 和 require 除了处理错误的方式不同之外，在其他方面都是相同的：
> require 生成一个致命错误（E_COMPILE_ERROR），在错误发生后脚本会停止执行。
> include 生成一个警告（E_WARNING），在错误发生后脚本会继续执行。

这个在小规模开发的时候，没什么大问题。但在大型的开发项目中，这么做会产生大量的require或者include方法调用，这样不因降低效率，而且使得代码难以维护，况且require_once的代价很大。
在PHP5之前，各个PHP框架如果要实现类的自动加载，一般都是按照某种约定自己实现一个遍历目录，自动加载所有符合约定规则的文件的类或函数。 当然，PHP5之前对面向对象的支持并不是太好，类的使用也没有现在频繁。 在PHP5后，当加载PHP类时，如果类所在文件没有被包含进来，或者类名出错，Zend引擎会自动调用__autoload 函数。此函数需要用户自己实现__autoload函数。 在PHP5.1.2版本后，可以使用spl_autoload_register函数自定义自动加载处理函数。当没有调用此函数，默认情况下会使用SPL自定义的spl_autoload函数。

### 2. php自动加载
### 3. composer自动加载
### 4. laravel自动加载
