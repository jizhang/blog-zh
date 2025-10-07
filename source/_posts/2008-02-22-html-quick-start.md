---
title: "HTML 扫盲教程"
date: 2008-02-22 19:03:00
categories: Programming
tags: [html, tutorial]
---

HTML 是 HyperText Markup Language 的缩写，中文名称为超文本标记语言。它实际上就是以 `.htm` 或 `.html` 为扩展名的文本文件，可以用记事本等编辑器直接编辑。下面让我们看看一个最基本的 HTML 文件。

```html
<html>
  <head>
    <title>问候</title>
  </head>
  <body>
    <h1>你好，世界！</h1>
  </body>
</html>
```

这便是经典的“你好世界”的例子了。在记事本里输入上述代码，然后保存为 greeting.htm 文件（一定要注意扩展名，可以打开资源管理器 — 工具 — 文件夹选项 - 查看，把“隐藏已知文件类型的扩展名”前的钩去掉，这样就可以查看并更改文件的扩展名了）随后双击该文件，默认 IE 就会打开此文件，并显示其中的内容。这个例子显示的是1号字体的“你好，世界！”。

普通的文本文件为何会有这种功能呢？关键就在于标签，即用尖括号 `<>` 括住的部分，如 `<html>`、`<h1>` 等。当 IE 打开该文件的时候会解释、编译这些标签，因为每个标签都有不同的含义。如 `<title>` 标签设置该网页的标题，`<h1>` 标签设置其中内容的字体大小等。

<!-- more -->

一个网页的基本结构是：

```html
<html>
  <head>...<title>...</title>...</head>
  <body>...</body>
</html>
```

其中：
* `<html>` 表示HTML文件的开始，`</html>` 表示结束
* `<head>` 表示头信息的开始，`<title>` 和 `</title>` 标签包含网页的标题，且该标签必须包含在头信息内，`</head>` 表示头信息结束
* `<body>` 表示网页内容的开始，需要显示的内容都要插在这部分内，`</body>` 表示结束

这里我们可以看到，HTML 的开始标签和结束标签必须成对出现，且不可交错。如 `<h1><h2>...</h1></h2>` 是不合法的。当然，有些 HTML 标签可以省略结束标签，如 `<img>`（插入图象）和 `<br>`（插入换行）等，但是在新出台的 XHTML 规范中，这些标签也是需要关闭的，方法是使用关闭标签如 `</img>`，或者直接使用 `<img src="mypic.jpg"/>`，但目前可以不考虑这些问题。

在 `<img src="mypic.jpg"/>` 中，`src` 是什么呢？它是 `<img>` 标签的属性，这里是用来指定所插入的图片的地址的。标签大多有各自特定的属性，如 `<body bgcolor="gray">` 就表示将网页的背景颜色设置为灰色。所以，学习 HTML 语言只要了解常用的 HTML 标签就可以了，下表将列出比较常用的 HTML 标签和他们各自的属性，以供学习和参考：

### 一、文本相关
1、标题标签 `<h1>`～`<h6>`

前面已经提到过，该系列的标签用来设置字体的大小。如 `<h3>这是三级标题</h3>`。

2、字体标签 `<font size="字号" color="颜色">`

该标签用以设置字体的大小和颜色。如 `<font size="5" color="red">红色的5号字</font>`：

<font size="5" color="red">红色的5号字</font>

`size` 的值可以是 1-7 之间的数字，`color` 可以是预定义的关键字，如 `red`、`blue` 等，也可以是十六进制颜色值，如 `#000000` 表示黑色，`#ffffff` 表示白色等。这些内容需另外学习。

3、特殊格式
* `<b>` <b>加粗</b>
* `<i>` <i>斜体字</i>
* `<u>` <u>下划线</u>
* `<strike>` <strike>删除线</strike>
* `<sup>` 这是<sup>上标</sup>
* `<sub>` 这是<sub>下标</sub>

4、分段标签 `<p align="left|center|right">`

`align` 属性表示段内文字的对齐方式。

<p align="right">这是右对齐的文字</p>

5、换行标签 `<br/>`

虽然在 HTML 的规范中可以直接用 `<br>`，但考虑到今后向 XHTML 转型，应该使用 `<br/>`，其他不需要关闭的标签也应该养成关闭的习惯。

6、移动文本标签

`<marquee direction="left|right|up|down" behavior="scroll|alternate|slide">`

`direction` 表示文字移动的方向，`behavior` 表示移动方式，具体各位可以自己试一试。

<marquee direction="right" behavior="alternate">这是从左向右来回移动的文本</marquee>

7、水平线标签 `<hr size="宽度" color="颜色" width="宽度"/>`

<hr>

### 二、插入图片

`<img src="地址" border="边框宽度"/>`

<img src="/zh/images/pydp-qrcode.jpg"/>

### 三、超级链接

`<a href="地址">链接</a>`

<a href="/zh/">张吉的数据笔记</a>

### 四、基本表格标签

先看一个基本的表格：

```html
<table border="1">
  <tr><th>字段1</th><th>字段2</th></tr>
  <tr><td>单元格1-1</td><td>单元格1-2</td></tr>
  <tr><td>单元格2-1</td><td>单元格2-2</td></tr>
</table>
```

<table>
  <tr><th>字段1</th><th>字段2</th></tr>
  <tr><td>单元格1-1</td><td>单元格1-2</td></tr>
  <tr><td>单元格2-1</td><td>单元格2-2</td></tr>
</table>

这是一个三行两列的表格。其中：
* `<table>` 标签表示一个表格的开始
* `<tr>` 标签表示表格的行
* `<th>` 标签表示表头单元格
* `<td>` 标签表示普通单元格

因为是简单教程所以很多东西不能讲得太细，否则就是一堆了。我编写这个教程的初衷是希望大家不要觉得网页制作很神秘，其实这些都很简单的，关键看你愿不愿意学了。
