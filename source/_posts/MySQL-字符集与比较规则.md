---
title: MySQL 字符集与比较规则
date: 2022-08-10 01:00:03
tags: MySQL
description: MySQL 字符集与比较规则的解析
---
## 1. 字符集解析：

总所周知，计算机智能存储二进制的数据，那么我们会不会好奇，数据库是如何将字符数据转换成二进制数据保存的呢？这时候我们就要引入字符集的概念。字符集将每一个字符都与一个二进制数据进行绑定。

举个例子，下面是 utf8 字符集中对应中文字符与二进制的映射关系。

| 字符 | 对应二进制                |
| ---- | ------------------------- |
| a    | 00000001 (十六进制：0x01) |
| b    | 00000010 (十六进制：0x02) |
| A    | 00000011 (十六进制：0x03) |
| B    | 00000100 (十六进制：0x04) |

如果我们想要将一个字符映射成一个二进制数据的过程叫做`编码`，反过来，如果将一个二进制数据映射成一个字符则叫做`解码`。

如果编码和解码使用的字符集不一致，就容易导致乱码，比如某个字节按照 utf8 编码成的数据值是 xxx, 但是如果按照 gbk 字符集解码，就可能会被映射成 gbk 字符集中的另一个字符。

## 2. 字符集比较规则

我们在开发中常常会用到排序，比如将某个列表按照首字母排序，比较规则顾名思义是字符集的比较规则，在比较字符串大小或者对某个字符串排序中用到。

# 二、MySQL 中的字符集与比较规则配置

# 1. MySQL中字符集和比较规则的级别

 ### MySQL 有四个级别的字符集和比较规则，分别是：

* 服务器级别
* 数据库级别
* 表级别
* 列级别

### 各个级别字符集和比较规则的设置

#### 服务器级别

服务器级别的字符集和比较规则在配置文件中设置，启动服务器时自动读取配置，配置文件写法：

```ini
[server]
character_set_server=gbk
collation_server=gbk_chinese_ci
```

#### 数据库级别

创建、修改数据库的时候可以指定数据库的字符集和比较规则，如果没有指定，则默认获取服务器默认的字符集和制定规则。

```ini
CREATE DATABASE 数据库名
    [[DEFAULT] CHARACTER SET 字符集名称]
    [[DEFAULT] COLLATE 比较规则名称];

ALTER DATABASE 数据库名
    [[DEFAULT] CHARACTER SET 字符集名称]
    [[DEFAULT] COLLATE 比较规则名称];
```

#### 表级别

创建、修改表的时候可以指定表的字符集和比较规则，如果没有指定，则获取数据库默认的字符集和制定规则。

```ini
CREATE TABLE 表名 (列的信息)
    [[DEFAULT] CHARACTER SET 字符集名称]
    [COLLATE 比较规则名称]]

ALTER TABLE 表名
    [[DEFAULT] CHARACTER SET 字符集名称]
    [COLLATE 比较规则名称]
```

#### 列级别

创建和修改列的时候可以指定列的字符集和比较规则，同一个表中的不同列可以使用不同字符集或比较规则，如果没有指定则会获取表默认的字符集和制定规则。

```ini
CREATE TABLE 表名(
    列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称],
    其他列...
);

ALTER TABLE 表名 MODIFY 列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称];
```

#### 注意：**字符集不一致有可能会导致索引失效**



## 2. 客户端和服务器通信中使用的字符集：

从客户端发送请求到服务器，服务器处理请求，最后返回结果给客户端，会经历三次编码解码的过程。这三次编码解码使用的字符集通过系统变量进行配置。

| 系统变量                 | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| character_set_client     | 服务器解码请求时使用的字符集                                 |
| character_set_connection | 服务器处理请求时会把请求字符串从`character_set_client`转为`character_set_connection` |
| character_set_results    | 服务器向客户端返回数据时使用的字符集                         |

**一个字符串进入服务器到返回的历程**

如果我把 character_set_client 和 character_set_results 的字符集定为 utf8, character_set_connection 字符集定为 gbk。当客户端发送请求，请求进入到服务器，根据服务器配置 character_set_client = utf8，会认为请求是 utf8 编码数据，则将 utf8 数据解码成指定的字符串，又因为 character_set_connection = gbk，服务器将指定字符串又编码成 gbk 数据。通过这个 gbk 数据查找到的结果会由于 配置character_set_results = utf8, 则又需要将结果解码成 utf8 字符，才能返回。

**容易造成的问题**

我们分析字符串从进入服务器到返回结果的过程中得知，如果客户端的编码格式与配置的服务器接受请求的字符集或服务器返回的字符集不一致，都有可能导致乱码。

**解决办法**

为了解决服务器编码解码过程中容易造成的乱码问题，我们通常都把 ***character_set_client*** 、***character_set_connection***、***character_set_results*** 这三个系统变量设置成和客户端使用的字符集一致的情况，这样减少了很多无谓的字符集转换。

我们有两种直接设置以上三个系统变量编码格式的方法：

```ini
SET NAMES 字符集名;
```

这一条语句产生的效果和我们执行这3条的效果是一样的：

```ini
SET character_set_client = 字符集名;
SET character_set_connection = 字符集名;
SET character_set_results = 字符集名;
```

第二种方法是写在配置文件里，启动服务器的时候直接配置好。
```ini
[client]
default-character-set=utf8
```
