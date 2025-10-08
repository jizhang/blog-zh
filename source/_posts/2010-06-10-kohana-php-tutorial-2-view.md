---
title: "Kohana PHP 教程[2]：视图（View）"
date: 2010-06-10 18:20:00
categories: [Translation]
tags: [php, kohana, tutorial]
---

欢迎来到 Kohana PHP 3.0 (KO3) 系列教程的第二部分。如果你还没有阅读第一部分的话请[点击这里](https://kohanaframework.org/)。本文将介绍如何使用 KO3 开发视图。

在开始之前，我们先来下载最新的 KO3 代码。点击[这个链接](http://dev.kohanaphp.com/projects/kohana3/files)，将下载到的文件解压到 kohana 或 mykohana3 的文件夹中。完成后，记得删除 install.php 文件。然后打开 application 目录下的 bootstrap.php 文件，将以下代码：

```php
Kohana::init(array('base_url' => '/kohana/'));
```

修改为：

```php
Kohana::init(array('base_url' => '/mykohana3/', 'index_file' => ''));
```

现在我们已经更新好的代码，下面我们就来尝试开发一个视图。在 application 目录下新建一个名为 views 的目录，在 views 目录下建立 pages 目录。打开文本编辑器，输入以下代码：

```html
<html>
  <head>
    <title>Hello!</title>
  </head>
  <body>
    <h1>This is my first view</h1>
  </body>
</html>
```

保存为 application/views/pages/ko3.php。很显然，这是一个非常简单 HTML 页面文件。打开控制器文件：application/classes/controller/ko3.php，用以下代码替换 `action_index()` 函数：

```php
public function action_index()
{
    $this->request->response = View::factory('pages/ko3');
}
```

保存文件，在浏览器中输入 http://yourserver/myfirstkohana3/ 这时你就应该能够看到“This is my first view”的字样。

<!-- more -->

上面的代码非常简单，使用 `View` 类的 `factory` 方法来加载 application/views/pages/ko3.php，经过处理后发送到客户端中。这似乎还不足以让人兴奋，那就让我们回到视图文件（application/views/pages/ko3.php），在 `<h1>` 标签后添加以下代码：

```php
<?php echo $content; ?>
```

这样，你的文件就应该如下所示：

```php
<html>
  <head>
    <title>Hello!</title>
  </head>
  <body>
    <h1>This is my first view</h1>
    <?php echo $content; ?>
  </body>
</html>
```

刷新浏览器页面，这时 KO3 会报错称使用了一个未定义的变量 `$content`。解决方法是在控制器中对视图所使用到的变量进行赋值，代码如下：

```php
public function action_index()
{
    $view = View::factory('pages/ko3');
    $view->content = 'We have data!';
    $this->request->response = $view->render();
}
```

再次刷新浏览器，就能看到“This is my first view”的下方会显示“We have data!”。下面让我们逐行解释这些代码。

```php
$view = View::factory('pages/ko3');
```

加载视图文件 application/views/pages/ko3.php 到视图控制器中。

```php
$view->content = 'We have data!';
```

对视图中所使用到的变量进行赋值。

```php
$this->request->response = $view->render();
```

处理视图，并输出到客户端。

很简单吧。还有另一种方法可以为变量赋值，代码如下：

```php
public function action_index()
{
    $data['content'] = 'We have data!';
    $view = View::factory('pages/ko3', $data);
    $this->request->response = $view->render();
}
```

简单地说，上面的代码使用了一个数组来传递变量。这就是全部了吗？别急，我们还有两种方法可以达到传递变量的目的。

```php
public function action_index()
{
    $view = View::factory('pages/ko3')->set('content', 'We have data!');
    $this->request->response = $view->render();
}
```

上面的代码使用了 `View` 类的 `set` 方法，并且可以采用链式代码来设置全部的变量。

```php
public function action_index()
{
    $content = 'We have data!';
    $view = View::factory('pages/ko3')->bind('content', $content);
    $this->request->response = $view->render();
}
```

第四种方法是使用 `View` 类的 `bind` 方法来传递变量，同样支持链式调用。但这种方法和前面几种有所不同，它传递给视图的是一个变量的引用。比如，在绑定 `$content` 时它的值为“We have data!”，但随后我们仍可以改变它。请看以下代码：

```php
public function action_index()
{
    $content = 'We have data!';
    $view = View::factory('pages/ko3')->bind('content', $content);
    $content = 'Our data changed';
    $this->request->response = $view->render();
}
```

浏览器中显示的是“Our data changed”，而不是之前的“We have data!”。

下面我们尝试在视图中调用视图。在 application/views/ 下新建 blocks 目录，并建立名为 ko3_inner.php 的文件，写入以下代码：

```html
<h3>This is an inner view</h3>
```

再次编辑 application/views/pages/ko3.php，加入以下代码：

```php
<?php echo View::factory('blocks/ko3_inner')->render(); ?>
```

使得最终文件内容如下所示：

```php
<html>
 <head>
  <title>Hello!</title>
 </head>
 <body>
  <h1>This is my first view</h1>
  <?php echo $content;?>
  <?php echo View::factory('blocks/ko3_inner')->render(); ?>
 </body>
</html>
```

刷新浏览器，我们就能看到多了一行“This is an inner view”的字样。这个功能在编写静态页面时非常有用，但是，我们不能在这个子视图中直接使用变量。那么下面就让我们解决这个问题。打开控制器（application/classes/controller/ko3.php），编辑 `action_index()` 的代码：

```php
public function action_index()
{
    $ko3_inner['content'] = 'We have more data';
    $ko3['content'] = 'We have data';
    $ko3['ko3_inner'] = View::factory('blocks/ko3_inner', $ko3_inner)->render();
    $view = View::factory('pages/ko3', $ko3);
    $this->request->response = $view->render();
}
```

这会使得主视图在被处理时同时处理子视图，从而使变量可用。可以注意到，这里我先处理了子视图，然后再处理主视图。这个顺序必须要注意。下面，我们编辑主视图的代码：

```php
<html>
 <head>
  <title>Hello!</title>
 </head>
 <body>
  <h1>This is my first view</h1>
  <?php echo $content; ?>
  <?php echo $ko3_inner; ?>
 </body>
</html>
```

再编辑子视图的代码：

```php
<h3>This is an inner view</h3>
<?php echo $content; ?>
```

这样，浏览器中就会显示如下：

```
This is my first view

We have data

This is an inner view

We have more data
```

很棒吧？这样你就可以让你的视图变得更模块化，便于重用。最后，我们学习如何让某个变量可以被所有的视图使用。在控制器中（application/classes/controller/ko3.php）中编辑 `action_index()` 方法：

```php
public function action_index()
{
    View::set_global('x', 'This is a global variable');
    $ko3_inner['content'] = 'We have more data';
    $ko3['content'] = 'We have data';
    $ko3['ko3_inner'] = View::factory('blocks/ko3_inner', $ko3_inner)->render();
    $view = View::factory('pages/ko3', $ko3);
    $this->request->response = $view->render();
}
```

然后在每个视图中插入如下代码：

```php
<br/><?php echo $x; ?>
```

刷新浏览器，这时你就能看到“This is a global variable”的字样出现了两次。简单地说，我们调用了 `View` 类的一个静态方法，使得所有的视图都能看到这个变量。你也可以用静态的 `bind` 方法来定义引用变量。

下一篇我将会介绍模板（template）的使用，谢谢。
