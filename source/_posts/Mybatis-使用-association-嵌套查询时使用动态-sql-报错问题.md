---
title: Mybatis 使用 association 嵌套查询时使用动态 sql 报错问题
date: 2020-08-27 17:58:12
tags: 笔记
description: 记一次嵌套查询使用错误
---
## 前情:
在项目中使用 association 作联合查询的时候发现报错，一直找不到原因。

#### 错误码：
> Caused by: org.apache.ibatis.reflection.ReflectionException: There is no getter for property named 'userId' in 'class java.lang.String'

#### 有问题的代码：
嵌套的resultMap:
```
<resultMap id="UserResult" type="com.jdf.user.web.model.User">
    <result column="user_id" property="user_id"/>
    <result column="user_ename" property="user_ename"/>
    <result column="user_name" property="user_name"/>
    <association property="position" column="user_id" select="getUserPosition"/>
</resultMap>

<select id="selectUseWithPosition" resultMap="UserResult">
    select user_id, user_ename, user_name from user
</select>

<resultMap id="PositionResult" type="com.jdf.user.web.model.Position">
    <result column="user_id" property="user_id"/>
    <result column="position" property="position"/>
</resultMap>

<select id="getUserPosition" resultMap="PositionResult">
    select user_id, position from position
    where 1 = 1
    <if test="user_id != null and user_id != ''">
        and user_id = #{user_id, jdbcType=VARCHAR}
    </if>
</select>
```
## 问题排查过程:
在排查问题时，通过错误码搜出来的博客，发现大部分人的报错原因都是多个入参没有加上 @Param 注解，导致映射出问题。最后参考了网友 [木之多](https://www.shuzhiduo.com/A/nAJv6rRxJr/) 的博客。

**原因：**
因为联合查询的效果相当于对每个入参都执行: select user_id, position from position where user_id = resultSet.getString("user_id") 语句。
不需要使用 <if test="user_id != null and user_id != ''"> 这种动态 sql 去判断是否有入参，association 的嵌套查询一定会有一个入参。

#### 修改后的代码:
将被嵌套查询的 getUserPosition sql 语句改为:
```
<select id="getUserPosition" resultMap="PositionResult">
    select user_id, position from position
    where user_id = #{user_id, jdbcType=VARCHAR}
    </if>
</select>
```

**最后:** 问题解决，希望能帮助到你。
ヽ(∀ﾟ )人(ﾟ∀ﾟ)人( ﾟ∀)人(∀ﾟ )人(ﾟ∀ﾟ)人( ﾟ∀)ﾉ
