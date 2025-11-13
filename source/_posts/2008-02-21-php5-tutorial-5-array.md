---
title: "PHP5 薄荷教程[5]：数组"
date: 2008-02-21 20:20:00
categories: [Programming]
tags: [php, tutorial]
---

## 数组的创建
### 1. `$arr = array([key1 =>] value1, [[key2 =>] value2, ...])`
其中 `$arr` 是数组名，`array()` 是空数组，其中的 `key1`、`key2` 分别是数组元素 `value1`、`value2` 的索引，它可以是整数或字符串，如果省略则默认为从 0 开始的整数列；数组元素可以是任意类型，包括数组。如：
```php
$arr = array(1, 2, 3);
$weekday = array(1 => "Monday", 2 => "Tuesday", 3 => "Wednesday", 4 => "Thursday");
$month = array("Jan" => "January", "Feb" => "February", "Mar" => "March", "Apr" => "April");
```

### 2. `$arr[key] = value`
```php
$weekday[1] = "Monday";
$month["Apr"] = "April";
```

### 3. `$arr[] = value`
如果数组不存在，则创建数组并以 0 为索引加入元素；如果数组已存在，则以数组中各索引的最大值加 1 作为新元素的索引。如：
```php
$arr[] = "0";  // $arr[0] = "0"
$arr[5] = "5";  // $arr[5] = "5"
$arr[] = "6";  // $arr[6] = "6"
```

## 数组的使用
用 `$arr[key]` 即可对数组中的某个元素进行读写操作。若要遍历数组中的所有元素，可以用 `foreach` 关键字，如：
```php
$arr = array("a", "b", "c", "d");
foreach ($arr as $key => $value) {
    print "Key: $key => Value: $value<br/>";
}
```
