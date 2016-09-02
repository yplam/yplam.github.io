---
title:  "PHP中的empty()与isset()"
categories: PHP
---

对PHP程序员而言，empty()与isset()肯定不会陌生，甚至是每天都会调用N百遍，正因为如此，我们每个人都认为对它们已经熟悉的不得了，甚至连文档都懒得再翻阅一下，前赴后继的往坑里冲。

我今天又掉坑里一次了，所以特意又翻阅了一遍文档：

**empty — 检查一个变量是否为空**

{% highlight php %}
bool empty ( mixed $var )
{% endhighlight %}

如果 var 是非空或非零的值，则 empty() 返回 FALSE。换句话说，""、0、"0"、NULL、FALSE、array()、var $var; 以及没有任何属性的对象都将被认为是空的，如果 var 为空，则返回 TRUE。

因为是一个语言构造器而不是一个函数，不能被 可变函数 调用。empty() 只检测变量，检测任何非变量的东西都将导致解析错误。换句话说，后边的语句将不会起作用： empty(addslashes($name))。

**isset — 检测变量是否设置**

{% highlight php %}
bool isset ( mixed $var [, mixed $... ] )
{% endhighlight %}

检测变量是否设置，并且不是 NULL。

如果已经使用 unset() 释放了一个变量之后，它将不再是 isset()。若使用 isset() 测试一个被设置成 NULL 的变量，将返回 FALSE。同时要注意的是一个 NULL 字节（"\0"）并不等同于 PHP 的 NULL 常数。

因为是一个语言构造器而不是一个函数，不能被 可变函数 调用

### **PHP 5.4 中empty()与isset()已经发生改变**

PHP 5.4 changes how empty() behaves when passed string offsets.

{% highlight php %}

$expected_array_got_string = 'somestring';
var_dump(empty($expected_array_got_string['some_key']));
var_dump(empty($expected_array_got_string[0]));
var_dump(empty($expected_array_got_string['0']));
var_dump(empty($expected_array_got_string[0.5]));
var_dump(empty($expected_array_got_string['0.5']));
var_dump(empty($expected_array_got_string['0 Mostel']));

{% endhighlight %}

以上例程在PHP 5.3中的输出：

{% highlight php %}
    bool(false)
    bool(false)
    bool(false)
    bool(false)
    bool(false)
    bool(false)
{% endhighlight %}

以上例程在PHP 5.4中的输出：

{% highlight php %}
    bool(true)
    bool(false)
    bool(false)
    bool(false)
    bool(true)
    bool(true)
{% endhighlight %}

PHP 5.4 changes how isset() behaves when passed string offsets.

{% highlight php %}

$expected_array_got_string = 'somestring';
var_dump(isset($expected_array_got_string['some_key']));
var_dump(isset($expected_array_got_string[0]));
var_dump(isset($expected_array_got_string['0']));
var_dump(isset($expected_array_got_string[0.5]));
var_dump(isset($expected_array_got_string['0.5']));
var_dump(isset($expected_array_got_string['0 Mostel']));

{% endhighlight %}


以上例程在PHP 5.3中的输出：

{% highlight php %}

    bool(true)
    bool(true)
    bool(true)
    bool(true)
    bool(true)
    bool(true)
{% endhighlight %}

以上例程在PHP 5.4中的输出：

{% highlight php %}
    bool(false)
    bool(true)
    bool(true)
    bool(true)
    bool(false)
    bool(false)

{% endhighlight %}
