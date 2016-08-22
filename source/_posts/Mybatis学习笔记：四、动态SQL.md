---
title: Mybatis学习笔记：四、动态SQL
date: 2016-06-03 13:46:20
tags: [Mybatis , 动态SQL]
---
## 简介
Mybatis另一个重要的特性就是动态SQL，利用这一特性，可以简化Java代码。
Mybatis的动态SQL主要包含以下几种语句：

- if
- choose (when, otherwise)
- trim (where, set)
- foreach

## if

```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’ 
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```
## choose,when,otherwise
choose有些类似于switch。
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```
## trim, where, set
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG 
  WHERE 
  <if test="state != null">
    state = #{state}
  </if> 
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```
考虑一个问题，当上面if代码段中，如果两个结果都为false，那么sql就变为：
```
SELECT * FROM BLOG 
  WHERE 
```
或者第一个为false，第二个为true，则是：
```
SELECT * FROM BLOG
WHERE 
AND title like ‘someTitle’
```
这样显然是有问题的，Mybatis简单的为我们解决了这一问题：
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG 
  <where> 
    <if test="state != null">
         state = #{state}
    </if> 
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
  </where>
</select>
```
`where`元素会在至少有一个`if`条件为true的情况下才会插入`WHERE`，也会知道如何去除`OR`|`AND`。

我们也能通过`trim`来解决这一问题，可以通过自定义`trim`来定制我们想要的功能。比如，上面的`where`元素等价于：
```
 <select id="queryUser" parameterType="java.lang.Integer" resultMap="BaseResultMap">
    SELECT * FROM users
    <trim prefix="WHERE" prefixOverrides="AND |OR ">
       AND id = #{id}
    </trim>
  </select>
```
`prefixOverrides`会自动忽略里面的`AND`，并且插入`prefix`中指定的内容`WHERE`，这样，执行时的SQL会变成：
`SELECT * FROM users WHERE id = #{id}`
用于动态更新的标签为`SET`
```
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
```
这里，`set`会动态前置`set`，并忽略无关的逗号。

## foreach
主要用于对集合的遍历:
```
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  WHERE ID in
  <foreach item="item" index="index" collection="mylist"
      open="(" separator="," close=")">
        #{item}
  </foreach>
</select>
```
其中item为每次迭代取到的值，index为迭代次数。
当collection是map时，index是键。

## bind
`bind`元素可以从 OGNL 表达式中创建一个变量并将其绑定到上下文：
```
<select id="selectBlogsLike" resultType="Blog">
  <bind name="pattern" value="'%' + _parameter.getTitle() + '%'" />
  SELECT * FROM BLOG
  WHERE title LIKE #{pattern}
</select>
```

## 多数据库支持
```
<insert id="insert">
  <selectKey keyProperty="id" resultType="int" order="BEFORE">
    <if test="_databaseId == 'oracle'">
      select seq_users.nextval from dual
    </if>
    <if test="_databaseId == 'db2'">
      select nextval for seq_users from sysibm.sysdummy1"
    </if>
  </selectKey>
  insert into users values (#{id}, #{name})
</insert>
```