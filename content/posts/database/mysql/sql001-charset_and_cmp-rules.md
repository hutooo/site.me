---
title: 字符集与比较规则
author: ash
tags: ["MySQL", "SQL", "DataBase"]
categories: ["数据库探秘", "Hacking"]
date: 2020-07-22T16:49:48+08:00
cover: "/images/miku01.jpg"
---

## 字符集与比较规则简介

### 字符集简介

众所周知，计算机只能存储二进制数据，`100110101010...`，那我们是怎么看到各式各样的字符的呢？当然是做映射啦~，这里要搞清楚两件事儿~

1. 需要将哪些字符映射成数据
2. 当然是如何映射

人们抽象出一个字符集的概念来描述某个字符范围的编码规则.
比方说我们来自定义一个名称为 `meow` 的字符集，它包含的字符范围和编码规则如下...

* 包含字符 `'a', 'A', 'b', 'B'`
* 采用一个字节，编码一个字符
  ```c
  'a' -> 00000001 // 0x01
  'b' -> 00000010 // 0x02
  'A' -> 00000011 // 0x03
  'B' -> 00000100 // 0x04
  ```

有了 `meow` 字符集， 我们就可以用二进制数表示字符了...
  ```c
  'bA'  -> 0000001000000011 // 0x0203
  'baB' -> 000000100000000100000100 // 0x020104
  'cd'  -> // 无法表示
  ```

### 比较规则简介

现在我们确定了 `meow` 字符集表示字符的范围以及编码规则，但是该怎么比较两个字符的大小呢？

最容易想到的就是直接比较这两个字符对应的二进制编码的大小，比方说字符'a'的编码为0x01，字符'b'的编码为0x02，所以'a'小于'b'，这种简单的比较规则也可以被称为二进制比较规则[binary collation]

二进制比较规则是简单，但有时候并不符合现实需求，比如在很多场合对于英文字符我们都是不区分大小写的，也就是说'a'和'A'是相等的，在这种场合下就不能简单粗暴的使用二进制比较规则了，这时候我们可以这样指定比较规则：

* 将两个大小写不同的字符全都转为大写或者小写
* 然后，再比较这两个字符对应的二进制数据

这是一种稍微复杂一点点的比较规则，但是实际生活中的字符不止英文字符一种，比如我们的汉字有几万之多，对于某一种字符集来说，比较两个字符大小的规则可以制定出很多种，也就是说同一种字符集可以有多种比较规则，我们稍后就要介绍各种现实生活中用的字符集以及它们的一些比较规则

### 一些重要的字符集

我们生活的世界太大了，为了满足不同的人的需要，制定出了多种字符集，它们表示的字符范围和用到的编码规则可能都不一样. 我们看一下一些常用字符集的情况...

* `ASCII` 字符集
* `ISO-8859-1` 字符集
* `GB2312` 字符集
* `GBK` 字符集
* `utf8` 字符集

> 小贴士：
其实准确的说，utf8只是Unicode字符集的一种编码方案，Unicode字符集可以采用utf8、utf16、utf32这几种编码方案，utf8使用1～4个字节编码一个字符，utf16使用2个或4个字节编码一个字符，utf32使用4个字节编码一个字符。更详细的Unicode和其编码方案的知识不是本书的重点，大家上网查查哈～
MySQL中并不区分字符集和编码方案的概念，所以后边唠叨的时候把utf8、utf16、utf32都当作一种字符集对待。

## MySQL中支持的字符集和排序规则

### utf8 与 utf8mb4

我们上边说utf8字符集表示一个字符需要使用1～4个字节，但是我们常用的一些字符使用1～3个字节就可以表示了

在MySQL中字符集表示一个字符所用最大字节长度在某些方面会影响系统的存储和性能，所以MySQL的设计者偷偷的定义了两个概念：

* utf8mb3：阉割版的utf8字符集，只使用1～3个字节表示字符

* utf8mb4：正宗的utf8字符集，使用1～4个字节表示字符

有一点需要大家十分的注意，在MySQL中 utf8 是 utf8mb3 的别名，所以之后在MySQL中提到 utf8 就意味着使用1~3个字节来表示一个字符，如果大家有使用4字节编码一个字符的情况，比如存储一些emoji表情啥的，那请使用utf8mb4

```sql
-- 查看字符集
-- SHOW (CHARACTER SET|CHARSET) [LIKE 匹配的模式];

mysql> SHOW CHARSET;
+----------+---------------------------------+---------------------+--------+
| Charset  | Description                     | Default collation   | Maxlen |
+----------+---------------------------------+---------------------+--------+
| big5     | Big5 Traditional Chinese        | big5_chinese_ci     |      2 |
...
| latin1   | cp1252 West European            | latin1_swedish_ci   |      1 |
| latin2   | ISO 8859-2 Central European     | latin2_general_ci   |      1 |
...
| ascii    | US ASCII                        | ascii_general_ci    |      1 |
...
| gb2312   | GB2312 Simplified Chinese       | gb2312_chinese_ci   |      2 |
...
| gbk      | GBK Simplified Chinese          | gbk_chinese_ci      |      2 |
| latin5   | ISO 8859-9 Turkish              | latin5_turkish_ci   |      1 |
...
| utf8     | UTF-8 Unicode                   | utf8_general_ci     |      3 |
...
| latin7   | ISO 8859-13 Baltic              | latin7_general_ci   |      1 |
| utf8mb4  | UTF-8 Unicode                   | utf8mb4_general_ci  |      4 |
| utf16    | UTF-16 Unicode                  | utf16_general_ci    |      4 |
| utf16le  | UTF-16LE Unicode                | utf16le_general_ci  |      4 |
...
| utf32    | UTF-32 Unicode                  | utf32_general_ci    |      4 |
| binary   | Binary pseudo charset           | binary              |      1 |
...
| gb18030  | China National Standard GB18030 | gb18030_chinese_ci  |      4 |
+----------+---------------------------------+---------------------+--------+
41 rows in set (0.01 sec)
```

```sql
-- 查看比较规则
-- SHOW COLLATION [LIKE 匹配的模式];

mysql> SHOW COLLATION LIKE 'utf8\_%';
+--------------------------+---------+-----+---------+----------+---------+
| Collation                | Charset | Id  | Default | Compiled | Sortlen |
+--------------------------+---------+-----+---------+----------+---------+
| utf8_general_ci          | utf8    |  33 | Yes     | Yes      |       1 |
| utf8_bin                 | utf8    |  83 |         | Yes      |       1 |
| utf8_unicode_ci          | utf8    | 192 |         | Yes      |       8 |
| utf8_icelandic_ci        | utf8    | 193 |         | Yes      |       8 |
| utf8_latvian_ci          | utf8    | 194 |         | Yes      |       8 |
| utf8_romanian_ci         | utf8    | 195 |         | Yes      |       8 |
| utf8_slovenian_ci        | utf8    | 196 |         | Yes      |       8 |
| utf8_polish_ci           | utf8    | 197 |         | Yes      |       8 |
| utf8_estonian_ci         | utf8    | 198 |         | Yes      |       8 |
| utf8_spanish_ci          | utf8    | 199 |         | Yes      |       8 |
| utf8_swedish_ci          | utf8    | 200 |         | Yes      |       8 |
| utf8_turkish_ci          | utf8    | 201 |         | Yes      |       8 |
| utf8_czech_ci            | utf8    | 202 |         | Yes      |       8 |
| utf8_danish_ci           | utf8    | 203 |         | Yes      |       8 |
| utf8_lithuanian_ci       | utf8    | 204 |         | Yes      |       8 |
| utf8_slovak_ci           | utf8    | 205 |         | Yes      |       8 |
| utf8_spanish2_ci         | utf8    | 206 |         | Yes      |       8 |
| utf8_roman_ci            | utf8    | 207 |         | Yes      |       8 |
| utf8_persian_ci          | utf8    | 208 |         | Yes      |       8 |
| utf8_esperanto_ci        | utf8    | 209 |         | Yes      |       8 |
| utf8_hungarian_ci        | utf8    | 210 |         | Yes      |       8 |
| utf8_sinhala_ci          | utf8    | 211 |         | Yes      |       8 |
| utf8_german2_ci          | utf8    | 212 |         | Yes      |       8 |
| utf8_croatian_ci         | utf8    | 213 |         | Yes      |       8 |
| utf8_unicode_520_ci      | utf8    | 214 |         | Yes      |       8 |
| utf8_vietnamese_ci       | utf8    | 215 |         | Yes      |       8 |
| utf8_general_mysql500_ci | utf8    | 223 |         | Yes      |       1 |
+--------------------------+---------+-----+---------+----------+---------+
27 rows in set (0.00 sec)
```