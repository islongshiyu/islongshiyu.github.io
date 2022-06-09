---
title: "分页算法"
date: 2022-05-09T14:33:12+08:00
lastmod: 2022-05-09T15:03:00+08:00
categories: ["Coding"]
---

分页算法在编程语言层面主要考虑一个情况，即总数 `rows` 是否能被页长 `length` 整除（换言之又回到了数学上的**取模运算**，求两个数相除的余数；有余数则页数为**商**加 `1`。）；文中的三种分页算法代码都是对这个情况使用不用的奇淫技巧进行处理。

<!--more-->

## 方式一
    
```java
// 总数
int rows = 101;
// 页长
int length = 20;
// 页数
int pages = (rows - 1) / length + 1;
```

这种方式的思想为只要 `rows` 多出 `1` 条数据，页数就多一页，多 `1` 条数据和 多 `length - 1` 条数据都是多 `1` 页；因此只需要考虑临界值情况即可，即恰好多一条的情况。

`rows` 的值为为 `81` 或 `99` 时，公式 `int pages = (rows - 1) / length + 1` 的结果都是一致的，因为多出来的数据没有大于**模**的值时是不会再多一页的，**模**即 `length`。 

## 方式二

```java
// 总数
int rows = 101;
// 页长
int length = 20;
// 页数
int pages = rows % length == 0 ? rows / length : rows / length + 1; 
```

使用取模运算来判断是否有余数，有余数则页数为 `rows / length + 1`，无余数则页数为 `rows / length`。

## 方式三

```java
// 总数
int rows = 101;
// 页长
int length = 20;
// 页数
int pages = (rows + length - 1) / length;
```

等同于方式一，只不过是对方式一采用数学层面的合并分子分母后的公式。