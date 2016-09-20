---
title: Mybatis学习笔记：二、Configuration
date: 2016-05-31 13:30:39
tags: Mybatis
---
## 简介
Configuration中包含了Mybatis的大部分配置信息，它为我们提供了设置的方法。下面是一个简单的配置样例。
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 对事务的管理和连接池的配置 -->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/test?characterEncoding=UTF-8" />
                <property name="username" value="root" />
                <property name="password" value="root" />
            </dataSource>
        </environment>
    </environments>

    <!-- mapping 文件路径配置 -->
    <mappers>
        <mapper resource="com/test/mappers/UserMapper.xml" />
    </mappers>
</configuration>
```

## 配置信息介绍
  Configuration配置文件中有以下属性可配置：

- properties：用于配置属性信息。

- settings：用于配置MyBatis的运行时方式。

- typeAliases：配置类型别名，可以在xml中用别名取代全限定名。

- typeHandlers：配置类型处理器。

- objectFactory：对象工厂。

- plugins：配置拦截器，用于拦截sql语句的执行。

- environments：配置数据源信息、连接池、事务属性等。

- mappers：配置SQL映射文件。

## 配置详解
1. properties
   properties用于定义配置文件中需要用到的属性值，它里面的属性将被存放在Configuration的variables变量里面，供Mybatis使用。

   配置实例：
   ```
	jdbc.properties:
	jdbc.dirverClassName=com.mysql.jdbc.Driver
	jdbc.url=jdbc:mysql://127.0.0.1:3306/test?characterEncoding=UTF-8
	jdbc.username=root
	jdbc.password=root
	jdbc.validationQuery=SELECT 1
   ```
   ```
	mybatis-config.xml:
	<properties resource="jdbc.properties">
        <property name="jdbc.username" value="root2"/>
    </properties>

	<property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
   ```
   属性也可以被传递到 SqlSessionBuilder.build()方法中。例如：
	```Java
	SqlSessionFactory factory = sqlSessionFactoryBuilder.build(reader, props);
	// ... or ...
	SqlSessionFactory factory = sqlSessionFactoryBuilder.build(reader, environment, props);
	```

 <font color=red>**注意：通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的是 properties 属性中指定的属性。**</font>

2. settings
   一个完整的settings元素的示例如下：
   ```
    <settings>
		<!--该配置影响的所有映射器中配置的缓存的全局开关 -->
        <setting name="cacheEnabled" value="true"/>
		<!--延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置fetchType属性来覆盖该项的开关状态。 -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!--当启用时，对任意延迟属性的调用会使带有延迟加载属性的对象完整加载；反之，每种属性将会按需加载。 -->
        <setting name="aggressiveLazyLoading" value="false"/>
        <!--是否允许单一语句返回多结果集（需要兼容驱动）。-->
        <setting name="multipleResultSetsEnabled" value="true"/>
        <!--使用列标签代替列名。不同的驱动在这方面会有不同的表现， 具体可参考相关驱动文档或通过测试这两种不同的模式来观察所用驱动的结果。-->
        <setting name="useColumnLabel" value="true"/>
        <!--允许 JDBC 支持自动生成主键，需要驱动兼容。 如果设置为 true 则这个设置强制使用自动生成主键，尽管一些驱动不能兼容但仍可正常工作（比如 Derby）。-->
        <setting name="useGeneratedKeys" value="false"/>
        <!--指定 MyBatis 应如何自动映射列到字段或属性。 NONE 表示取消自动映射；PARTIAL 只会自动映射没有定义嵌套结果集映射的结果集。 FULL 会自动映射任意复杂的结果集（无论是否嵌套）。 -->
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <!--当自动映射检测到未知的列时，会表现什么行为。NONE什么都不做，WARNING输出警告日志（日志中'org.apache.ibatis.session.AutoMappingUnknownColumnBehavior'需要设为warn），FAILING抛出SqlSessionException-->
        <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
        <!--配置默认的执行器。SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新。-->
        <setting name="defaultExecutorType" value="SIMPLE"/>
        <!--设置超时时间，它决定驱动等待数据库响应的秒数-->
        <setting name="defaultStatementTimeout" value="25"/>
        <!--设置默认全局查询记录条数，在query中的设置会覆盖此属性-->
        <setting name="defaultFetchSize" value="100"/>
        <!--允许在嵌套语句中使用分页（RowBounds）。如果允许，设为false-->
        <setting name="safeRowBoundsEnabled" value="false"/>
        <!--是否开启自动驼峰命名规则（camel case）映射，即从经典数据库列名 A_COLUMN 到经典 Java 属性名 aColumn 的类似映射。-->
        <setting name="mapUnderscoreToCamelCase" value="false"/>
        <!--MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。-->
        <setting name="localCacheScope" value="SESSION"/>
        <!--当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。 -->
        <setting name="jdbcTypeForNull" value="OTHER"/>
        <!--指定哪个对象的方法触发一次延迟加载。-->
        <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
    </settings>
   ```
3. typeAliases
   类型别名是为 Java 类型设置一个短的名字。它只和 XML 配置有关，存在的意义仅在于用来减少类完全限定名的冗余.
   ```
    <typeAliases>
      <typeAlias alias="Author" type="domain.blog.Author"/>
      <typeAlias alias="Blog" type="domain.blog.Blog"/>
      <typeAlias alias="Comment" type="domain.blog.Comment"/>
      <typeAlias alias="Post" type="domain.blog.Post"/>
      <typeAlias alias="Section" type="domain.blog.Section"/>
      <typeAlias alias="Tag" type="domain.blog.Tag"/>
    </typeAliases>    
   ```
   也可以指定一个包名，MyBatis 会在包名下面搜索需要的 Java Bean，比如:
   ```
	<typeAliases>
	    <package name="domain.blog"/>
	</typeAliases>
   ```
4. typeHandlers
   无论是 MyBatis 在预处理语句（PreparedStatement）中设置一个参数时，还是从结果集中取出一个值时， 都会用类型处理器将获取的值以合适的方式转换成 Java 类型。Mybatis内置了常见类型的转换器。也允许自定义自己的转换器来处理Mybatis不支持的数据类型。
   具体做法是实现`org.apache.ibatis.type.TypeHandler`接口，或者继承`org.apache.ibatis.type.BaseTypeHandler`类。

    ```Java
    /**
     * 将object映射到数据库的varchar类型
     */
    @MappedJdbcTypes(JdbcType.VARCHAR)
    public class MyTypeHandler extends BaseTypeHandler<Object> {
        @Override
        public void setNonNullParameter(PreparedStatement preparedStatement, int i, Object o, JdbcType jdbcType) throws SQLException {
    
        }
    
        @Override
        public Object getNullableResult(ResultSet resultSet, String s) throws SQLException {
            return null;
        }
    
        @Override
        public Object getNullableResult(ResultSet resultSet, int i) throws SQLException {
            return null;
        }
    
        @Override
        public Object getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        return null;
    }
   }
   ```
   ```
      <!-- mybatis-config.xml -->
      <typeHandlers>
        <typeHandler handler="org.mybatis.example.ExampleTypeHandler"/>
      </typeHandlers>
   ```
5. objectFactory
   默认的对象工厂需要做的仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。 如果想覆盖对象工厂的默认行为，则可以通过创建自己的对象工厂来实现。如下：
   ```Java
     public class MyObjectFactory extends DefaultObjectFactory {
         public Object create(Class type) {
             return super.create(type);
         }
         public Object create(Class type, List<Class> constructorArgTypes, List<Object> constructorArgs) {
    return super.create(type, constructorArgTypes, constructorArgs);
         }
         public void setProperties(Properties properties) {
             super.setProperties(properties);
         }
         public <T> boolean isCollection(Class<T> type) {
             return Collection.class.isAssignableFrom(type);
         }
     }
   ```
   ```
     <!-- mybatis-config.xml -->
     <objectFactory type="org.mybatis.example.MyObjectFactory">
       <property name="someProperty" value="100"/>
     </objectFactory>
   ```
   ObjectFactory 接口很简单，它包含两个创建用的方法，一个是处理默认构造方法的，另外一个是处理带参数的构造方法的。 最后，setProperties 方法可以被用来配置 ObjectFactory，在初始化你的 ObjectFactory 实例后， objectFactory 元素体中定义的属性会被传递给 setProperties 方法。
6. plugins
   Mybatis允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，Mybatis允许拦截的方法包括：
   - Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed) 
   - ParameterHandler (getParameterObject, setParameters) 
   - ResultSetHandler (handleResultSets, handleOutputParameters)
   - StatementHandler (prepare, parameterize, batch, update, query)
    	
   ```Java
	// ExamplePlugin.java
      @Intercepts({@Signature(
        type= Executor.class,
        method = "update",
        args = {MappedStatement.class,Object.class})})
      public class ExamplePlugin implements Interceptor {
        public Object intercept(Invocation invocation) throws Throwable {
          return invocation.proceed();
        }
        public Object plugin(Object target) {
          return Plugin.wrap(target, this);
        }
        public void setProperties(Properties properties) {
        }
      }

      <!-- mybatis-config.xml -->
      <plugins>
        <plugin interceptor="org.mybatis.example.ExamplePlugin">
          <property name="someProperty" value="100"/>
        </plugin>
      </plugins>
   ```
  
 上面的插件将会拦截在 Executor 实例中所有的 “update” 方法调用， 这里的 Executor 是负责执行低层映射语句的内部对象。 

7. environments
   每个 SqlSessionFactory 实例只能选择一个environment，每个数据库对应一个SqlSessionFactory 实例 。
   - 事务管理器
       - JDBC – 这个配置就是直接使用了 JDBC 的提交和回滚设置，它依赖于从数据源得到的连接来管理事务范围。
       - MANAGED – 这个配置几乎没做什么。它从来不提交或回滚一个连接，而是让容器来管理事务的整个生命周期（比如 JEE 应用服务器的上下文）。 默认情况下它会关闭连接，然而一些容器并不希望这样，因此需要将 closeConnection 属性设置为 false 来阻止它默认的关闭行为。例如: 
       ```
		<transactionManager type="MANAGED">
           <property name="closeConnection" value="false"/>
        </transactionManager>
       ```
   如果使用的是spring容器来管理Mybatis，则不需要配置事务管理器，因为spring会使用自带的事务管理器来覆盖之前的配置。
   - 数据源
       - UNPOOLED - 每次被请求时打开和关闭连接。
         
          - driver – 这是 JDBC 驱动的 Java 类的完全限定名（并不是JDBC驱动中可能包含的数据源类）。
          - url – 这是数据库的 JDBC URL 地址。
          - username – 登录数据库的用户名。
          - password – 登录数据库的密码。
          - defaultTransactionIsolationLevel – 默认的连接事务隔离级别。

       - POOLED - 使用数据库连接池。
         
         - poolMaximumActiveConnections – 在任意时间可以存在的活动（也就是正在使用）连接数量，默认值：10
         - poolMaximumIdleConnections – 任意时间可能存在的空闲连接数。
         - poolMaximumCheckoutTime – 在被强制返回之前，池中连接被检出（checked out）时间，默认值：20000 毫秒（即 20 秒）
         - poolTimeToWait – 这是一个底层设置，如果获取连接花费的相当长的时间，它会给连接池打印状态日志并重新尝试获取一个连接（避免在误配置的情况下一直安静的失败），默认值：20000 毫秒（即 20 秒）。
         - poolPingQuery – 发送到数据库的侦测查询，用来检验连接是否处在正常工作秩序中并准备接受请求。默认是“NO PING QUERY SET”，这会导致多数数据库驱动失败时带有一个恰当的错误消息。
         - poolPingEnabled – 是否启用侦测查询。若开启，也必须使用一个可执行的 SQL 语句设置 poolPingQuery 属性（最好是一个非常快的 SQL），默认值：false。
         - poolPingConnectionsNotUsedFor – 配置poolPingQuery 的使用频度。这可以被设置成匹配具体的数据库连接超时时间，来避免不必要的侦测，默认值：0（即所有连接每一时刻都被侦测 — 当然仅当 poolPingEnabled 为 true 时适用）。
         
       - JNDI - 这个数据源的实现是为了能在如 EJB 或应用服务器这类容器中使用
       
8. mappers
   
   配置映射文件位置：
   ```
     <!-- Using classpath relative resources -->
     <mappers>
       <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
       <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
       <mapper resource="org/mybatis/builder/PostMapper.xml"/>
     </mappers>
  
	<!-- Using url fully qualified paths -->
      <mappers>
        <mapper url="file:///var/mappers/AuthorMapper.xml"/>
        <mapper url="file:///var/mappers/BlogMapper.xml"/>
        <mapper url="file:///var/mappers/PostMapper.xml"/>
      </mappers>
   
	<!-- Using mapper interface classes -->
      <mappers>
        <mapper class="org.mybatis.builder.AuthorMapper"/>
        <mapper class="org.mybatis.builder.BlogMapper"/>
        <mapper class="org.mybatis.builder.PostMapper"/>
      </mappers>
   
    <!-- Register all interfaces in a package as mappers -->
      <mappers>
        <package name="org.mybatis.builder"/>
      </mappers>
   ```