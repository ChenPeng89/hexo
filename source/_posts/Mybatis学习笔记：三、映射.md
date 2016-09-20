---
title: Mybatis学习笔记：三、映射
date: 2016-05-31 19:04:10
tags: Mybatis 
---

## 简介
Mybatis的映射文件包括以下几个顶级元素：
  - cache – 给定命名空间的缓存配置。
  - cache-ref – 其他命名空间缓存配置的引用。
  - resultMap – 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。
  - parameterMap – 已废弃！
  - sql – 可被其他语句引用的可重用语句块。
  - insert – 映射插入语句
  - update – 映射更新语句
  - delete – 映射删除语句
  - select – 映射查询语句


## 详细配置
### Select
   使用样例：
   ```
	  <select id="selectPerson" parameterType="int" resultType="hashmap">
			SELECT * FROM PERSON WHERE ID = #{id}
	  </select>
   ```
   这个语句被称作 selectPerson，接受一个 int（或 Integer）类型的参数，并返回一个 HashMap 类型的对象，其中的键是列名，值便是结果行中的对应值。
   `#{id}`告诉Mybatis创建一个preparedStatement，并用id来替换预处理sql中的?。
   ```Java
	   // Similar JDBC code, NOT MyBatis…
       String selectPerson = "SELECT * FROM PERSON WHERE ID=?";
       PreparedStatement ps = conn.prepareStatement(selectPerson);
       ps.setInt(1,id);
   ```
   
   Select详细配置
   ```
     <select
       <!--在命名空间中唯一的标识符，可以被用来引用这条语句。-->
       id="selectPerson"
       <!--将会传入这条语句的参数类的完全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过 TypeHandler 推断出具体传入语句的参数，默认值为 unset。-->
       parameterType="int"
       <!--从这条语句中返回的期望类型的类的完全限定名或别名。注意如果是集合情形，那应该是集合可以包含的类型，而不能是集合本身。使用 resultType 或 resultMap，但不能同时使用。-->
       resultType="hashmap"
       <!--外部 resultMap 的命名引用。结果集的映射是 MyBatis 最强大的特性，对其有一个很好的理解的话，许多复杂映射的情形都能迎刃而解。使用 resultMap 或 resultType，但不能同时使用。 -->
       resultMap="personResultMap"
       <!--将其设置为 true，任何时候只要语句被调用，都会导致本地缓存和二级缓存都会被清空，默认值：false。 -->
       flushCache="false"
       <!--将其设置为 true，将会导致本条语句的结果被二级缓存，默认值：对 select 元素为 true。-->
       useCache="true"
       <!--这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为 unset（依赖驱动）。-->
       timeout="10000"
       <!--这是尝试影响驱动程序每次批量返回的结果行数和这个设置值相等。默认值为 unset（依赖驱动）。-->
       fetchSize="256"
       <!--STATEMENT，PREPARED 或 CALLABLE 的一个。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 -->
       statementType="PREPARED"
       <!--FORWARD_ONLY，SCROLL_SENSITIVE 或 SCROLL_INSENSITIVE 中的一个，默认值为 unset （依赖驱动）。 -->
       resultSetType="FORWARD_ONLY">
   ```
### insert,update,delete
   insert，update 和 delete 的实现非常接近：
   ```
	  <insert
      <!--命名空间中的唯一标识符，可被用来代表这条语句。-->
      id="insertAuthor"
      <!--将要传入语句的参数的完全限定类名或别名。这个属性是可选的，因为 MyBatis 可以通过 TypeHandler 推断出具体传入语句的参数，默认值为 unset。 -->
      parameterType="domain.blog.Author"
      <!--将其设置为 true，任何时候只要语句被调用，都会导致本地缓存和二级缓存都会被清空，默认值：true（对应插入、更新和删除语句）。 -->
      flushCache="true"
      <!--STATEMENT，PREPARED 或 CALLABLE 的一个。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 -->
      statementType="PREPARED"
      <!--（仅对 insert 和 update 有用）唯一标记一个属性，MyBatis 会通过 getGeneratedKeys 的返回值或者通过 insert 语句的 selectKey 子元素设置它的键值，默认：unset。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。 -->
      keyProperty=""
      <!--（仅对 insert 和 update 有用）通过生成的键值设置表中的列名，这个设置仅在某些数据库（像 PostgreSQL）是必须的，当主键列不是表中的第一列的时候需要设置。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。 -->
      keyColumn=""
      <!--仅对 insert 和 update 有用）这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系数据库管理系统的自动递增字段），默认值：false。 -->
      useGeneratedKeys=""
      这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为 unset（依赖驱动）。-->
      timeout="20">
      
      <update
        id="updateAuthor"
        parameterType="domain.blog.Author"
        flushCache="true"
        statementType="PREPARED"
        timeout="20">
      
      <delete
        id="deleteAuthor"
        parameterType="domain.blog.Author"
        flushCache="true"
        statementType="PREPARED"
        timeout="20">
   ```
	
	
  如果你的数据库支持自动生成主键的字段（比如 MySQL 和 SQL Server），那么你可以设置 useGeneratedKeys=”true”，然后再把 keyProperty 设置到目标属性上就OK了。例如，如果 Author 表已经对 id 使用了自动生成的列类型，那么为:
	
   ```
    <insert id="insertAuthor" useGeneratedKeys="true"
    keyProperty="id">
    	insert into Author (username,password,email,bio)values (#{username},#{password},#{email},#{bio})
    </insert>
   ```
   如果数据库支持多行插入，那么你可以传一个Authors的List当做参数
   ```
	<insert id="insertAuthor" useGeneratedKeys="true" keyProperty="id">
      insert into Author (username, password, email, bio) values
      <foreach item="item" collection="list" separator=",">
         (#{item.username}, #{item.password}, #{item.email}, #{item.bio})
      </foreach>
    </insert>
   ```
   对于不支持自动生成类型的数据库或可能不支持自动生成主键 JDBC 驱动来说，MyBatis 有另外一种方法来生成主键(当然这个不推荐。。。)。
   ```
     <insert id="insertAuthor">
        <selectKey keyProperty="id" resultType="int" order="BEFORE">
           select CAST(RANDOM()*1000000 as INTEGER) a from SYSIBM.SYSDUMMY1
        </selectKey>
       insert into Author (id, username, password, email,bio, favourite_section) values(#{id}, #{username}, #{password}, #{email}, #{bio}, #{favouriteSection,jdbcType=VARCHAR})
     </insert>
   ```
### selectKey ：
   ```
    <selectKey
      <!--selectKey 语句结果应该被设置的目标属性。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。-->
      keyProperty="id"
      <!--结果的类型。MyBatis 通常可以推算出来，但是为了更加确定写上也不会有什么问题。MyBatis 允许任何简单类型用作主键的类型，包括字符串。如果希望作用于多个生成的列，则可以使用一个包含期望属性的 Object 或一个 Map。 -->
      resultType="int"
      <!--这可以被设置为 BEFORE 或 AFTER。如果设置为 BEFORE，那么它会首先选择主键，设置 keyProperty 然后执行插入语句。如果设置为 AFTER，那么先执行插入语句，然后是 selectKey 元素 - 这和像 Oracle 的数据库相似，在插入语句内部可能有嵌入索引调用。 -->
      order="BEFORE"
      <!--与前面相同，MyBatis 支持 STATEMENT，PREPARED 和 CALLABLE 语句的映射类型，分别代表 PreparedStatement 和 CallableStatement 类型。-->
      statementType="PREPARED">
   ```
   
### sql:
   用来定义可重用的 SQL 代码段
   ```
    <sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
   ```
   这个 SQL 片段可以被包含在其他语句中，例如：
   ```
    <select id="selectUsers" resultType="map">
     select
       <include refid="userColumns"><property name="alias" value="t1"/></include>,
       <include refid="userColumns"><property name="alias" value="t2"/></include>
     from some_table t1
     cross join some_table t2
    </select>
   ```
### 参数
   `#{age,javaType=int,jdbcType=NUMERIC,,numericScale=2 ,typeHandler=MyTypeHandler}`

### ResultMap
   ResultMap是Mybatis中最重要最强大的元素，用来将数据库中的列映射到pojo相应字段上。
  
   一般来说，我们按照上面的示例：
   ```
      <select id="selectUsers" resultType="map">
         select id, username, hashedPassword
         from some_table
         where id = #{id}
      </select>
   ```
   这样，最终查询的结果会映射到一个hashmap中。如果我们想将它映射到User对象中，那么可以通过：
   ```
    <select id="selectUsers" resultType="xxx.xxx.User">
         select id, username, hashedPassword
         from some_table
         where id = #{id}
     </select>
   ```
   其中的原理，就是Mybatis创建一个ResultMap，然后将列映射到User上。如果select中的列名和User的属性名不一样，那么可能就不能进行匹配。就需要
   `select _id as id`
   这样的语句。
   这时候我们可以用resultMap来做映射，类似于hibernate的bean的映射文件的作用。
   ```
     <resultMap id="userResultMap" type="User">
          <id property="id" column="user_id" />
          <result property="username" column="user_name"/>
          <result property="password" column="hashed_password"/>
      </resultMap>
   ```
   这样在select时还可以按照原来的`as id`这样的语句就可以了。
   ```
       <select id="selectUsers" resultMap="userResultMap">
         select user_id, user_name, hashed_password
         from some_table
         where id = #{id}
       </select>
   ```
   接下来看一个复杂的ResultMap:
   ```
       <resultMap id="detailedBlogResultMap" type="Blog">
         <constructor>
           <idArg column="blog_id" javaType="int"/>
         </constructor>
         <result property="title" column="blog_title"/>
         <association property="author" javaType="Author">
           <id property="id" column="author_id"/>
           <result property="username" column="author_username"/>
            <result property="password" column="author_password"/>
            <result property="email" column="author_email"/>
            <result property="bio" column="author_bio"/>
            <result property="favouriteSection" column="author_favourite_section"/>
          </association>
          <collection property="posts" ofType="Post">
            <id property="id" column="post_id"/>
            <result property="subject" column="post_subject"/>
            <association property="author" javaType="Author"/>
         <collection property="comments" ofType="Comment">
           <id property="id" column="comment_id"/>
         </collection>
         <collection property="tags" ofType="Tag" >
           <id property="id" column="tag_id"/>
         </collection>
         <discriminator javaType="int" column="draft">
           <case value="1" resultType="DraftPost"/>
         </discriminator>
       </collection>
     </resultMap>
   ```
   分析一下，
   - `constructor ` --  类在实例化时,用来注入结果到构造方法中。
	   - `idArg` -- id
	   - `arg` -- 注入到构造方法中的参数 
	
   - `association ` -- 一对多关联
   - `discriminator ` -- 根据结果值来确定使用哪个结果集
	  - `case ` -- 类似于switch中的case。
	
   
   Constructor
   ```
        public class User {
           //...
           public User(int id, String username) {
             //...
          }
        //...
        }
   
       <constructor>
          <idArg column="id" javaType="int"/>
          <arg column="username" javaType="String"/>
       </constructor>
   ```
   在Mybatis中，支持使用构造方法注入属性，但是使用构造方法，需要保证参数和class中你定义的构造方法顺序是一样的，且id，type都是能够对应的。

   Association
   ```
       <association property="author" column="blog_author_id" javaType="Author">
         <id property="id" column="author_id"/>
         <result property="username" column="author_username"/>
       </association>
   ```
   Association表示的是一对一，一对多的关联，一般用于关联查询和嵌套查询中。
   ```
     <resultMap id="blogResult" type="Blog">
       <id property="id" column="blog_id" />
       <result property="title" column="blog_title"/>
       <association property="author" javaType="Author">
         <id property="id" column="author_id"/>
         <result property="username" column="author_username"/>
          <result property="password" column="author_password"/>
          <result property="email" column="author_email"/>
          <result property="bio" column="author_bio"/>
        </association>
      </resultMap>
   ```
   Collection
   ```
      <resultMap id="blogResult" type="Blog">
        <id property="id" column="blog_id" />
        <result property="title" column="blog_title"/>
        <collection property="posts" ofType="Post">
          <id property="id" column="post_id"/>
          <result property="subject" column="post_subject"/>
          <result property="body" column="post_body"/>
        </collection>
      </resultMap>
   ```
   discriminator
   ```
      <resultMap id="vehicleResult" type="Vehicle">
        <id property="id" column="id" />
        <result property="vin" column="vin"/>
        <result property="year" column="year"/>
        <result property="make" column="make"/>
        <result property="model" column="model"/>
        <result property="color" column="color"/>
        <discriminator javaType="int" column="vehicle_type">
          <case value="1" resultMap="carResult"/>
          <case value="2" resultMap="truckResult"/>
          <case value="3" resultMap="vanResult"/>
          <case value="4" resultMap="suvResult"/>
        </discriminator>
      </resultMap>
	```
   自动映射
   通常数据库列使用大写单词命名，单词间用下划线分隔；而java属性一般遵循驼峰命名法。 为了在这两种命名方式之间启用自动映射，需要将 `mapUnderscoreToCamelCase`设置为true。
   自动映射有三个级别：
   `NONE ` -- 不进行自动映射，需要手动设置属性
   `PARTIAL ` -- 只映射简单的，不映射嵌套的。
   `FULL ` --  会自动映射任意复杂的结果
   Mybatis默认的是`PARTIAL`。

### 缓存
   Mybatis提供了一个很强大的缓存机制。默认情况下二级缓存是关闭的。可以通过在SQL映射文件上添加 `<cache/>`来开启。
   这个简单语句的效果如下:

   - 映射语句文件中的所有 select 语句将会被缓存。
   - 映射语句文件中的所有 insert,update 和 delete 语句会刷新缓存。
   - 缓存会使用 Least Recently Used(LRU,最近最少使用的)算法来收回。
   - 根据时间表(比如 no Flush Interval,没有刷新间隔), 缓存不会以任何时间顺序 来刷新。
   - 缓存会存储列表集合或对象(无论查询方法返回什么)的 1024 个引用。
   - 缓存会被视为是 read/write(可读/可写)的缓存,意味着对象检索不是共享的,而且可以安全地被调用者修改,而不干扰其他调用者或线程所做的潜在修改。    

  还可以通过手动来设置它的属性：
	```
     <cache
       eviction="FIFO"
       flushInterval="60000" <!--每隔60s刷新-->
       size="512"
       readOnly="true"/>
	```
  回收策略包括：
   
   - LRU  – 最近最少使用的:移除最长时间不被使用的对象。
   - FIFO – 先进先出:按对象进入缓存的顺序来移除它们。
   - SOFT – 软引用:移除基于垃圾回收器状态和软引用规则的对象。
   - WEAK – 弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象。

  默认是LRU。

  readOnly(只读)属性可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓存对象的相同实例。因此这些对象不能被修改。这提供了很重要的性能优势。可读写的缓存会返回缓存对象的拷贝(通过序列化) 。这会慢一些,但是安全,因此默认是 false。
   
   