# 一.入门

## 安装

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>x.x.x</version>
</dependency>
```

## 构建SqlSessionFactory

> 方式1：

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

> 方式2

```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

## 从SqlSessionFactory中获取SqlSession

> 方式1

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
}
```

> 方式2

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
}
```

## 作用域和生命周期

**SqlSessionFactoryBuilder**：方法作用域

SqlSessionFactory：应用作用域，单例模式

SqlSession：线程不安全的，方法作用域或请求作用域

映射器实例：方法作用域

# 二.全局配置

* configuration（配置）
  * properties（属性）
  * settings（设置）
  * typeAliases（类型别名）
  * typeHandlers（类型处理器）
  * objectFactory（对象工厂）
  * plugins（插件）
  * environments（环境）
    * environment（环境变量）
      * transactionManager（事务管理器）
      * dataSource（数据源）
  * databaseIdProvider（数据库厂商标识）
  * mappers（映射器）

## 属性

> 属性方式1

```xml
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

> 属性方式2

```java
new SqlSessionFactoryBuilder().build(inputStream,props)
```

通过方法传递的props具有最高优先级，其次是 resource/url引入的配置文件，最低优先级则是properties中指定的元素

## 设置(settings)

| 设置名                            | <span style="display:inline-block;width: 70px"> 描述</span>  | 有效值                                                       | <span style="display:inline-block;width: 80px"> 默认值</span> | <span style="display:inline-block;width: 120px">疑惑</span>  |
| --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| cacheEnabled                      | 全局性打开或关闭所有映射器配置文件中已配置的任何缓存         | true\|false                                                  | true                                                         | 如果是true则开启二级缓存（使用CacheExecitor），且当mapper中配置了<cache>标签才会生效（添加缓存） |
| lazyLoadingEnabled                | 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。特定关联关系中可以通过设置fetchType属性覆盖该项开关状态 | true\|false                                                  | false                                                        | 在MappedStatement中，判断是否开启延迟加载会先查询fetchType属性，再根据Configuration中的lazyLoadingEnabled |
| aggressiveLazyLoading             | 开启方法时任一方法的调用都会加载对象的所有延迟加载属性，否则每个延迟加载属性会按需加载(参考lazyLoadTriggerMethods) | true\|false                                                  | false(3.4.1前版本默认为true)                                 | 如果aggressiveLazyLoading为true或者lazyLoadTriggerMethods包含调用方法时，会加载该对象的所有延迟属性 |
| multipleResultSetsEnabled         | 是否允许单个语句返回多个结果                                 | true\|false                                                  | true                                                         | 存储过程时可能会有这种情况                                   |
| useColumnLabel                    | 使用列标签代替列名                                           | true\|false                                                  | true                                                         | 不理解                                                       |
| useGeneratedKeys                  | 允许JDBC支持自动生成主键，如果设置为true，将强制使用自动生成主键 | true\|false                                                  | false                                                        | MappedStatement中的useGeneratedKeys的默认值                  |
| autoMappingBehavior               | 指定 MyBatis 应如何自动映射列到字段或属性。 NONE 表示关闭自动映射；PARTIAL 只会自动映射没有定义嵌套结果映射的字段。 FULL 会自动映射任何复杂的结果集（无论是否嵌套）。 | MONE\|PARTIAL\|FULL                                          | PARTIAL                                                      |                                                              |
| autoMappingUnknown ColumnBehavior | 指定发现自动映射目标未知列（或未知属性类型）的行为。                      `NONE`: 不做任何反应 `WARNING`: 输出警告日志（`'org.apache.ibatis.session .AutoMappingUnknown ColumnBehavior'` 的日志等级必须设置为 `WARN`） `FAILING`: 映射失败 (抛出 `SqlSessionException`) | NONE, WARNING, FAILING                                       | NONE                                                         |                                                              |
| defaultExecutorType               | 配置默认的执行器，SIMPLE-普通执行器，REUSE-执行器会重用预处理语句(PreparedStatement);BATCH-执行器不仅会重用语句，还会批量更新 | SIMPLE REUSE BATCH                                           | SIMPLE                                                       |                                                              |
| defaultStatementTimeout           | 设置超时时间，他决定数据库驱动等待数据库响应的秒数           |                                                              | 未设置                                                       |                                                              |
| defaultFetchSize                  | 为驱动的结果集获取数量                                       |                                                              | 未设置                                                       |                                                              |
| defaultResultSetType              | 指定语句默认的滚动策略。（新增于 3.5.2）                     | FOREARD_ONLY, SCROLL_SENSITIVE ,SCROLL_INSENSITIVE,DEFAULT   | 未设置                                                       |                                                              |
| mapUnderscoreTo CamelCase         | 是否开启驼峰命名自动映射，即从经典数据库列名 A_COLUMN 映射到经典 Java 属性名 aColumn | true,false                                                   | fasle                                                        |                                                              |
| localCacheScope                   | MyBatis 利用本地缓存机制（Local Cache）防止循环引用和加速重复的嵌套查询。 默认值为 SESSION，会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地缓存将仅用于执行语句，对相同 SqlSession 的不同查询将不会进行缓存。 | SESSION,STATEMENT                                            | SESSION                                                      |                                                              |
| jdbcTypeForNull                   | 当没有为参数指定特定的 JDBC 类型时，空值的默认 JDBC 类型。 某些数据库驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。 | JdbcType 常量，常用值： NULL、VARCHAR 或 OTHER。             | OTHER                                                        |                                                              |
| lazyLoadTriggerMethods            | 指定对象的哪些方法触发一次延迟加载                           | 用逗号分隔的方法列表。                                       | equals,clone, hashCode,toString                              |                                                              |
| defaultEnumTypeHandler            | 指定 Enum 使用的默认 `TypeHandler` 。（新增于 3.4.5）        | 一个类型别名或全限定类名                                     | org.apache .ibatis.type .EnumTypeHandler                     |                                                              |
| callSettersOnNulls                | 指定当结果集中值为 null 的时候是否调用映射对象的 setter（map 对象时为 put）方法，这在依赖于 Map.keySet() 或 null 值进行初始化时比较有用。注意基本类型（int、boolean 等）是不能设置成 null 的。 | true,false                                                   | false                                                        |                                                              |
| logPrefix                         | 指定 MyBatis 增加到日志名称的前缀。                          | 任何字符串                                                   | 未设置                                                       |                                                              |
| logImpl                           | 指定 MyBatis 所用日志的具体实现，未指定时将自动查找。        | SLF4J ,LOG4J ,LOG4J2,JDK_LOGGING, COMMONS_LOGGING, STOUT_LOGGING, NO_LOGGING | 未设置                                                       | LOG4J2                                                       |
| proxyFactory                      | 指定 Mybatis 创建可延迟加载对象所用到的代理工具              | CGLIB,JAVASSIST                                              | JAVASSIST                                                    |                                                              |
| vfsImpl                           | 指定 VFS 的实现                                              | 自定义 VFS 的实现的类全限定名,以逗号分隔。                   | 未设置                                                       |                                                              |
| useActualParamName                | 允许使用方法签名中的名称作为语句参数名称。 为了使用该特性，你的项目必须采用 Java 8 编译，并且加上 `-parameters` 选项。（新增于 3.4.1） | true,false                                                   | true                                                         |                                                              |
| configurationFactory              | 指定一个提供 `Configuration` 实例的类。 这个被返回的 Configuration 实例用来加载被反序列化对象的延迟加载属性值。 这个类必须包含一个签名为`static Configuration getConfiguration()` 的方法。（新增于 3.2.3） | 一个类型别名或完全限定类名                                   | 未设置                                                       | 当未设置时，反序列化对象的延迟加载属性并未生效               |

## 类型别名(typeAliases)

```xml
<typeAliases>
    <typeAlias alias="Author" type="domain.blog.Author"/>
</typeAliases>
<typeAliases>
    <package name= "domain.blog">
</typeAliases>
```

```java
@Alias("author")
public class Author {
    ...
}
```

## 类型处理器

> 实现 TypeHandler接口，或者继承 BaseTypeHandler类

```java
public class ExampleTypeHandler extends BaseTypeHandler<String> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, parameter);
  }

  @Override
  public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
    return rs.getString(columnName);
  }

  @Override
  public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    return rs.getString(columnIndex);
  }

  @Override
  public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    return cs.getString(columnIndex);
  }
}
```

```xml
<typeHandlers>
  <typeHandler handler="org.mybatis.example.ExampleTypeHandler" javaType="String" jdbcType="VARCHAR"/>
</typeHandlers>

<typeHandlers>
  <package name="org.mybatis.example"/>
</typeHandlers>
```

通过类型处理器的泛型，MyBatis 可以得知该类型处理器处理的 Java 类型，不过这种行为可以通过两种方法改变：

- 在类型处理器的配置元素（typeHandler 元素）上增加一个 `javaType` 属性（比如：`javaType="String"`）；
- 在类型处理器的类上增加一个 `@MappedTypes` 注解指定与其关联的 Java 类型列表。 如果在 `javaType` 属性中也同时指定，则注解上的配置将被忽略。

可以通过两种方式来指定关联的 JDBC 类型：

- 在类型处理器的配置元素上增加一个 `jdbcType` 属性（比如：`jdbcType="VARCHAR"`）；
- 在类型处理器的类上增加一个 `@MappedJdbcTypes` 注解指定与其关联的 JDBC 类型列表。 如果在 `jdbcType` 属性中也同时指定，则注解上的配置将被忽略。

当在 `ResultMap` 中决定使用哪种类型处理器时，此时 Java 类型是已知的（从结果类型中获得），但是 JDBC 类型是未知的。 因此 Mybatis 使用 `javaType=[Java 类型], jdbcType=null` 的组合来选择一个类型处理器。 这意味着使用 `@MappedJdbcTypes` 注解可以*限制*类型处理器的作用范围，并且可以确保，除非显式地设置，否则类型处理器在 `ResultMap` 中将不会生效。 如果希望能在 `ResultMap` 中隐式地使用类型处理器，那么设置 `@MappedJdbcTypes` 注解的 `includeNullJdbcType=true` 即可。 然而从 Mybatis 3.4.0 开始，如果某个 Java 类型**只有一个**注册的类型处理器，即使没有设置 `includeNullJdbcType=true`，那么这个类型处理器也会是 `ResultMap` 使用 Java 类型时的默认处理器。

> 因此一般会一个java类型只会有一个类型处理器

```xml
<!-- ResultMap中也可以指定特定的类型处理器 -->
<result column="roundingMode" property="roundingMode" typeHandler="org.apache.ibatis.type.EnumTypeHandler"/>
```

## 对象工厂(ObjectFactory)

mybatis创建结果对象新实例时会通过ObjectFactory来完成实例化工作

```java
public class ExampleObjectFactory extends DefaultObjectFactory {
  @Override
  public <T> T create(Class<T> type) {
    return super.create(type);
  }

  @Override
  public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return super.create(type, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
    super.setProperties(properties);
  }

  @Override
  public <T> boolean isCollection(Class<T> type) {
    return Collection.class.isAssignableFrom(type);
  }}
```

```xml
<!-- mybatis-config.xml -->
<objectFactory type="org.mybatis.example.ExampleObjectFactory">
  <property name="someProperty" value="100"/>
</objectFactory>
```

ObjectFactory 接口很简单，它包含两个创建实例用的方法，一个是处理默认无参构造方法的，另外一个是处理带参数的构造方法的。 另外，setProperties 方法可以被用来配置 ObjectFactory，在初始化你的 ObjectFactory 实例后， objectFactory 元素体中定义的属性会被传递给 setProperties 方法。

### 插件（plugins）

MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)

这些类中方法的细节可以通过查看每个方法的签名来发现，或者直接查看 MyBatis 发行包中的源代码。 如果你想做的不仅仅是监控方法的调用，那么你最好相当了解要重写的方法的行为。 因为在试图修改或重写已有方法的行为时，很可能会破坏 MyBatis 的核心模块。 这些都是更底层的类和方法，所以使用插件的时候要特别当心。

通过 MyBatis 提供的强大机制，使用插件是非常简单的，只需实现 Interceptor 接口，并指定想要拦截的方法签名即可。

```
// ExamplePlugin.java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  private Properties properties = new Properties();

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    // implement pre processing if need
    Object returnObject = invocation.proceed();
    // implement post processing if need
    return returnObject;
  }

  @Override
  public void setProperties(Properties properties) {
    this.properties = properties;
  }
}
<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

上面的插件将会拦截在 Executor 实例中所有的 “update” 方法调用， 这里的 Executor 是负责执行底层映射语句的内部对象。

> 构建Executor时，会将原始对象target，Intercept 封装到 Plugin对象中，而Plugin对象是实现InvocationHandler接口，可以通过 jdk动态代理产生新的代理对象
>
> 形成一个代理链模式

## 环境配置 (environments)

mybatis可以配置多个环境，但是每个SqlSessionFactory实例只能选择一种环境

```xml
<environments default="development">
    <environment id = "development">
    	<transactionManager type="JDBC">
            <property name ="..." value="..."/>
        </transactionManager>
        <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
          <property name="url" value="${url}"/>
          <property name="username" value="${username}"/>
          <property name="password" value="${password}"/>
        </dataSource>
    </environment>
</environments>
```

**事务管理器(transactionManager)**

在Mybatis中支持两种事务管理器（type=JDBC|MANAGED）

* JDBC-这个配置直接使用了JDBC的提交和回滚功能，默认在commit时会提交事务，但是在某些程序来说是不需要自动提交的因此可以配置 `???? 没太明白 连接不是关闭时自动提交事务吗`

  ```xml
  <transactionManager type="JDBC">
      <property name="skipSetAutoCommitOnClose" value="true"/>
  </transactionManager>
  ```

* MANAGED 这个配置的事务管理器，这个配置几乎没做什么，他从不提交或回滚一个连接，而是让容器管理事务的整个生命周期。默认情况下他会关闭连接，然而一些容器不希望连接被关闭，那么可以配置

  ```xml
  <transactionManager type="MANAGED">
    <property name="closeConnection" value="false"/>
  </transactionManager>
  ```

**数据源（dataSource）**

dataSource 元素使用标准的 JDBC 数据源接口来配置 JDBC 连接对象的资源。

- 大多数 MyBatis 应用程序会按示例中的例子来配置数据源。虽然数据源配置是可选的，但如果要启用延迟加载特性，就必须配置数据源。

有三种内建的数据源类型（也就是 type="[UNPOOLED|POOLED|JNDI]"）：

**UNPOOLED**– 这个数据源的实现会每次请求时打开和关闭连接。虽然有点慢，但对那些数据库连接可用性要求不高的简单应用程序来说，是一个很好的选择。 性能表现则依赖于使用的数据库，对某些数据库来说，使用连接池并不重要，这个配置就很适合这种情形。UNPOOLED 类型的数据源仅仅需要配置以下 5 种属性：

- `driver` – 这是 JDBC 驱动的 Java 类全限定名（并不是 JDBC 驱动中可能包含的数据源类）。
- `url` – 这是数据库的 JDBC URL 地址。
- `username` – 登录数据库的用户名。
- `password` – 登录数据库的密码。
- `defaultTransactionIsolationLevel` – 默认的连接事务隔离级别。
- `defaultNetworkTimeout` – 等待数据库操作完成的默认网络超时时间（单位：毫秒）。查看 `java.sql.Connection#setNetworkTimeout()` 的 API 文档以获取更多信息。

作为可选项，你也可以传递属性给数据库驱动。只需在属性名加上“driver.”前缀即可，例如：

- `driver.encoding=UTF8`

这将通过 DriverManager.getConnection(url, driverProperties) 方法传递值为 `UTF8` 的 `encoding` 属性给数据库驱动。

**POOLED**– 这种数据源的实现利用“池”的概念将 JDBC 连接对象组织起来，避免了创建新的连接实例时所必需的初始化和认证时间。 这种处理方式很流行，能使并发 Web 应用快速响应请求。

除了上述提到 UNPOOLED 下的属性外，还有更多属性用来配置 POOLED 的数据源：

- `poolMaximumActiveConnections` – 在任意时间可存在的活动（正在使用）连接数量，默认值：10
- `poolMaximumIdleConnections` – 任意时间可能存在的空闲连接数。
- `poolMaximumCheckoutTime` – 在被强制返回之前，池中连接被检出（checked out）时间，默认值：20000 毫秒（即 20 秒）
- `poolTimeToWait` – 这是一个底层设置，如果获取连接花费了相当长的时间，连接池会打印状态日志并重新尝试获取一个连接（避免在误配置的情况下一直失败且不打印日志），默认值：20000 毫秒（即 20 秒）。
- `poolMaximumLocalBadConnectionTolerance` – 这是一个关于坏连接容忍度的底层设置， 作用于每一个尝试从缓存池获取连接的线程。 如果这个线程获取到的是一个坏的连接，那么这个数据源允许这个线程尝试重新获取一个新的连接，但是这个重新尝试的次数不应该超过 `poolMaximumIdleConnections` 与 `poolMaximumLocalBadConnectionTolerance` 之和。 默认值：3（新增于 3.4.5）
- `poolPingQuery` – 发送到数据库的侦测查询，用来检验连接是否正常工作并准备接受请求。默认是“NO PING QUERY SET”，这会导致多数数据库驱动出错时返回恰当的错误消息。
- `poolPingEnabled` – 是否启用侦测查询。若开启，需要设置 `poolPingQuery` 属性为一个可执行的 SQL 语句（最好是一个速度非常快的 SQL 语句），默认值：false。
- `poolPingConnectionsNotUsedFor` – 配置 poolPingQuery 的频率。可以被设置为和数据库连接超时时间一样，来避免不必要的侦测，默认值：0（即所有连接每一时刻都被侦测 — 当然仅当 poolPingEnabled 为 true 时适用）。

**JNDI** – 这个数据源实现是为了能在如 EJB 或应用服务器这类容器中使用，容器可以集中或在外部配置数据源，然后放置一个 JNDI 上下文的数据源引用。这种数据源配置只需要两个属性：

- `initial_context` – 这个属性用来在 InitialContext 中寻找上下文（即，initialContext.lookup(initial_context)）。这是个可选属性，如果忽略，那么将会直接从 InitialContext 中寻找 data_source 属性。
- `data_source` – 这是引用数据源实例位置的上下文路径。提供了 initial_context 配置时会在其返回的上下文中进行查找，没有提供时则直接在 InitialContext 中查找。

和其他数据源配置类似，可以通过添加前缀“env.”直接把属性传递给 InitialContext。比如：

- `env.encoding=UTF8`

这就会在 InitialContext 实例化时往它的构造方法传递值为 `UTF8` 的 `encoding` 属性。

你可以通过实现接口 `org.apache.ibatis.datasource.DataSourceFactory` 来使用第三方数据源实现：

```java
public interface DataSourceFactory {
  void setProperties(Properties props);
  DataSource getDataSource();
}
```

`org.apache.ibatis.datasource.unpooled.UnpooledDataSourceFactory` 可被用作父类来构建新的数据源适配器，比如下面这段插入 C3P0 数据源所必需的代码：

```java
import org.apache.ibatis.datasource.unpooled.UnpooledDataSourceFactory;
import com.mchange.v2.c3p0.ComboPooledDataSource;

public class C3P0DataSourceFactory extends UnpooledDataSourceFactory {

  public C3P0DataSourceFactory() {
    this.dataSource = new ComboPooledDataSource();
  }
}
```

为了令其工作，记得在配置文件中为每个希望 MyBatis 调用的 setter 方法增加对应的属性。 下面是一个可以连接至 PostgreSQL 数据库的例子：

```xml
<dataSource type="org.myproject.C3P0DataSourceFactory">
  <property name="driver" value="org.postgresql.Driver"/>
  <property name="url" value="jdbc:postgresql:mydb"/>
  <property name="username" value="postgres"/>
  <property name="password" value="root"/>
</dataSource>
```

## 数据库厂商标识（databaseIdProvider）

MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 `databaseId` 属性。 MyBatis 会加载带有匹配当前数据库 `databaseId` 属性和所有不带 `databaseId` 属性的语句。 如果同时找到带有 `databaseId` 和不带 `databaseId` 的相同语句，则后者会被舍弃。 为支持多厂商特性，只要像下面这样在 mybatis-config.xml 文件中加入 `databaseIdProvider` 即可：

```xml
<databaseIdProvider type="DB_VENDOR" />
```

databaseIdProvider 对应的 DB_VENDOR 实现会将 databaseId 设置为 `DatabaseMetaData#getDatabaseProductName()` 返回的字符串。 由于通常情况下这些字符串都非常长，而且相同产品的不同版本会返回不同的值，你可能想通过设置属性别名来使其变短：

```xml
<databaseIdProvider type="DB_VENDOR">
  <property name="SQL Server" value="sqlserver"/>
  <property name="DB2" value="db2"/>
  <property name="Oracle" value="oracle" />
</databaseIdProvider>
```

在提供了属性别名时，databaseIdProvider 的 DB_VENDOR 实现会将 databaseId 设置为数据库产品名与属性中的名称第一个相匹配的值，如果没有匹配的属性，将会设置为 “null”。 在这个例子中，如果 `getDatabaseProductName()` 返回“Oracle (DataDirect)”，databaseId 将被设置为“oracle”。

你可以通过实现接口 `org.apache.ibatis.mapping.DatabaseIdProvider` 并在 mybatis-config.xml 中注册来构建自己的 DatabaseIdProvider：

```java
public interface DatabaseIdProvider {
  default void setProperties(Properties p) { // 从 3.5.2 开始，该方法为默认方法
    // 空实现
  }
  String getDatabaseId(DataSource dataSource) throws SQLException;
}
```

> 在解析 mapper.xml时，会只加载 databaseId为以上设置的或者为null的 select|insert|update|delete标签

## 映射器（mappers）

既然 MyBatis 的行为已经由上述元素配置完了，我们现在就要来定义 SQL 映射语句了。 但首先，我们需要告诉 MyBatis 到哪里去找到这些语句。 在自动查找资源方面，Java 并没有提供一个很好的解决方案，所以最好的办法是直接告诉 MyBatis 到哪里去找映射文件。 你可以使用相对于类路径的资源引用，或完全限定资源定位符（包括 `file:///` 形式的 URL），或类名和包名等。例如：

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>
<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>
<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```

这些配置会告诉 MyBatis 去哪里找映射文件，剩下的细节就应该是每个 SQL 映射文件了，也就是接下来我们要讨论的。

# 三.XML映射文件

SQL映射标签顶层元素

* `cache`-该命名空间的缓存配置，当xml引入该标签时，构建MappedStatement时，才会放入构建二级缓存对象
* `cache-ref`-引用其他命名空间的缓存配置
* `resultMap`-描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素
* ~~`parameterMap`-老式风格的参数映射，此元素已被废弃~~
* `sql`-可被其他语句引用的可重用语句块
* `select`-映射查询语句
* `update`-映射更新语句
* `insert`-映射插入语句
* `delete`-映射删除语句

## select



```xml
<select
        id="selectPerson"
        parameterType="int"
        parmeterMap="deprecated"
        resultType="hashMap"
        resultMap="personResultMap"
        flushCache="false"
        useCache="true"
        timeout="10"
        fetchSize="256"
        statementType="PREPARED"
        resultSetType="FORWARD_ONLY"
        >
</select>
```

| 属性                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| id                     | 命名空间的唯一标识,sqlSession.select("namespace.id")         |
| parameterType          | 传入语句参数的类全限定名或别名，属性可选，因为mybatis可以通过TypeHandler推断出传入语句参数，默认值未设置 |
| ~~parmeterMap~~        | ~~用于引用外部parameterMap属性，已废弃~~                     |
| resultType             | 期望从语句返回结果的类全限定名或别名，如果返回的是集合，则应该设置集合包含的类型，resultType和resultMap只能用一个 |
| resultMap              | 对外部resultMap的命名引用。resultType和resultMap只能用一个   |
| flushCache             | 将其设置为true后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false,如果设置为true，那么二级缓存和本地缓存都会更新 |
| useCache               | 将其设置为true后，将会导致本条语句结果被二级缓存缓存起来，默认值 select为true，如果设置useCache,mybatis会先从二级缓存中查询数据，并且将查询结果放入二级缓存 |
| timeout                | 这个设置是在抛出异常前，驱动程序等待数据库返回请求结果行数的秒数。默认值未未设置(依赖驱动程序)，mybatis会在Statement中设置timeout |
| fecthSize              | 这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个值。默认值为未设置，mysqltis会在Statement中设置fetchSize |
| statementType          | 可选 STATEMENT,PREPARED,CALLABLE。这会让Mybatis使用Statement,PrepatedStatement，CallAbleStatement，默认值：PREPARED |
| resultSetType          | FORWARD_ONLY,SCROLL_SENSITIVE,SCROLL_INSENSITIVE,DEFAULT(等价于未设置) |
| databaseId             | 如果配置了数据库厂商标识(databaseIdProvider)，Mybatis会加载不带有databaseId或当前databaseId的语句，如果带有和不带有都存在，则不带的会被忽略 |
| <u>`resultOrdered`</u> | 这个结果仅针对嵌套结果Select语句：如果为true,则假设结果集以正确的顺序(排序后) 执行映射，将不再发生对以前结果行的引用，这样可以减少内存消耗，默认值:false |
| resultSets             | 这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称以逗号分隔 |

## insert，update，delete

```xml
<insert
        id="insertAuthor"
        parameterType="domain.blog.Author"
        flushCache="true"
        statementType="PREPARED"
        keyProperty=""
        keyColumn=""
        useGeneratedKeys=""
        timeout="20">
</insert>
<update
        id="updateAuthor"
        parameterType="domain.nlog.Author"
        flushCache="true"
        statementType="PREPARED"
        timepout="20">
</update>
<delete
        id="deleteAuthor"
        parameterType="domain.blog.Author"
        statementType="PREPARED"
        timeout="20">
</delete>
```

| 属性             | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| flushCache       | 将其设置为true后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：(对insert，udpate，delete)true |
| useGeneratedKeys | （仅适用于insert和update）这会令Mybatis使用JDBC的getGeneratedKeys方法取出由数据库内部生成的主键，并且放入 parameter参数对应的属性内 |
| keyProperty      | （仅适用于insert和update）指定能够唯一识别对象的属性，Mybatis会使用getGeneratedKeys的返回值或insert语句中的selectKey子元素设置它的值，默认值：未设置unset，如果生成的列不知一个，可以用逗号分隔属性名称 |
| keyColumn        | （仅适用于insert和update）设置生成键值在表中的列名，在某些数据库中（PostgreSQL），当主键列不是表中的第一列时，是必须设置的，可以用逗号分隔多个属性名称 |
| databaseId       | 如果配置了数据库厂商标识，Mybatis会加载所有不带databaseId或匹配当前databaseId的语句，如果都存在则不带的会被忽略 |

```xml
<selectKey
           keyProperty="id"
           resultType="int"
           order="BEFORE"
           statementType="PREPARED">
</selectKey>
```

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| keyProperty | selectKey语句结果应该被设置到的目标属性。如果生成的列不止一个，可以用逗号分隔 |
| keyColumn   | 返回结果集中生成列属性的列名，如果生成列不止一个，可以用逗号分隔 |
| resultType  | 结果的类型                                                   |
| order       | 可以设置为BEFORE或AFTER，如果设置为BEFORE，那么它会首先生成主键，设置keyProperty再执行插入语句，如果设置为AFTER，那么会先执行插入语句，然后是selectKey中的语句-这和Oracle数据库行为类似 |

## sql

这个元素可以用来定义可重用的SQL代码片段。参数可以静态地（在加载xml时确定下来），并且可以在不同的include元素中定义不同的参数值

```xml
<sql id="userColumns" ${alias}.id,${alias}.username,${alias}.password></sql>
```

这个片段可以如下引用

```xml
<select id="selectUsers" resultType="map">
 select 
    <include refid="userColumns"> <property name="alias" value="t1"></include>
    <include refid="userColumns"> <property name="alias" value="t2"></include>
   from table1 t1
      inner join table2 t2 on xxx=xxx
</select>
```

## 参数

简单类型的参数会根据参数类型(ParameterType) 自动设置，这个参数可以随意命名。原始类型或简单简单数据类型，因为其没有属性，会用他们的值作为参数

```xml
<select id ="selectUser" resultType="User">
	select id,username,password
    from users
    <!-- #当中id可以随意设置-->
    where id =#{id} 
</select>
```

复杂类型的参数

```xml
<insert id="insertUser" parameterType="User">
  insert into users (id, username, password)
  values (#{id}, #{username}, #{password})
</insert>
```

## 字符串替换

默认情况使用 `#{}`参数语法时，Mybatis会创建`preparedStatement`参数占位符，并设置为? 。不过有时候你就是想直接在SQL语句中直接插入一个不转义的字符串。比如ORDER BY时，可以这么使用

```xml
ORDER BY ${columnName}
```

## 结果映射

### 高级结果映射

复杂的映射语句

```xml
<!-- 非常复杂的语句 -->
<select id="selectBlogDetails" resultMap="detailedBlogResultMap">
  select
       B.id as blog_id,
       B.title as blog_title,
       B.author_id as blog_author_id,
       A.id as author_id,
       A.username as author_username,
       A.password as author_password,
       A.email as author_email,
       A.bio as author_bio,
       A.favourite_section as author_favourite_section,
       P.id as post_id,
       P.blog_id as post_blog_id,
       P.author_id as post_author_id,
       P.created_on as post_created_on,
       P.section as post_section,
       P.subject as post_subject,
       P.draft as draft,
       P.body as post_body,
       C.id as comment_id,
       C.post_id as comment_post_id,
       C.name as comment_name,
       C.comment as comment_text,
       T.id as tag_id,
       T.name as tag_name
  from Blog B
       left outer join Author A on B.author_id = A.id
       left outer join Post P on B.id = P.blog_id
       left outer join Comment C on P.id = C.post_id
       left outer join Post_Tag PT on PT.post_id = P.id
       left outer join Tag T on PT.tag_id = T.id
  where B.id = #{id}
</select>
```

你可能想把它映射到一个智能的对象模型，这个对象表示了一篇博客，它由某位作者所写，有很多的博文，每篇博文有零或多条的评论和标签。 我们先来看看下面这个完整的例子，它是一个非常复杂的结果映射（假设作者，博客，博文，评论和标签都是类型别名）。 不用紧张，我们会一步一步地来说明。虽然它看起来令人望而生畏，但其实非常简单。

```xml
<!-- 非常复杂的结果映射 -->
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

### 结果映射（resultMap）

* `constructor`-用于在实例化类时，注入结果到构造方法中
  * `idArg`-ID参数；标记出作为ID的结果可以帮助提高整体性能
  * `arg`-将被注入到构造方法的一个普通结果
* `id`-一个ID结果；标记处作为ID的结果可以帮助提高整体性能
* `result`-注入到字段或javaBean属性的普通结果
* `assocation`-一个复杂的关联；许多结果将包装成这种类型
  * 嵌套结果映射-关联可以使`resultMap`元素，或是对其他结果的引用
* `collection`-一个复杂类型的集合
  * 嵌套结果映射-集合可以使`resultMap`元素，或是对其他结果的引用
* `discriminator`-使用结果值来决定使用哪个`resultMap`
  * `case`-基于某些值的结果映射
    * 嵌套结果映射-`case`也是一个结果映射，因此具有相同的结构和元素；或是引用其他结果映射

ResultMap的属性列表

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `id`          | 当前命名空间中的唯一标识，用于标识一个结果映射               |
| `type`        | 类的完全限定名或一个类型别名                                 |
| `autoMapping` | 如果设置这个属性，Mybatis将会为本结果映射开启或关闭自动映射，这个属性会覆盖全局的属性autoMappingBehavior。默认值未设置 |

#### id&result

```xml
<id property="id" column="post_id"/>
<result property="subject" column="post_subject"/>
```

这些元素是结果映射的基础。id和result元素都将一个列的值映射到一个简单数据类型的属性或字段

这两者之间的唯一不同是,id元素对用的属性会被标记为对象的标识符，在比较对象实例时使用，这样可以提高整体的性能，尤其是进行缓存和嵌套结果映射的时候

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `property`    | 映射到列结果的字段或属性，如果JavaBean有这个名字的属性(property)，会先使用该属性。否则Mybatis将会寻找给定字段名称的字段(field)。可以使用常见的点式分隔形式进行复杂属性导航，比如简单属性 "username"，复杂属性"address.street.number" |
| `column`      | 数据库当中的列名或别名，一般情况下这和传递给`resultSet.getString(columnName)方法参数一样` |
| `javaType`    | 一个Java类的全限定名，或一个类型别名。如果你映射到的是JavaBean,Mybatis可以推断类型，如果你映射的是HashMap，那么你应该指定JavaType来保证行为与期望的一致性 |
| `jdbcType`    | JDBC类型，只需要在可能执行插入、更新、删除的且允许空值列上指定JDBC类型 |
| `typeHandler` | 类型处理器，可以覆盖全局类型处理器                           |

#### 构造方法

通过可以修改对象属性的方式，可以满足大多数的数据传输对象，以及绝大部分领域模型的要求，但是有时候想使用不可变类，构造方法注入允许你在初始化时为类设置属性的值，而不暴露出公有方法

```java
public class User {
   //...
   public User(Integer id, String username, int age) {
     //...
  }
//...
}
```

为了将结果注入构造方法，需要在resultMap中采用 `constructor`标签，按照构造参数顺序映射

```xml
<constructor>
	<idArg column="id" javaType="int"/>
    <arg column="username" javaType="String"/>
    <arg column="age" javaType="_int"/>
</constructor>
```

从版本3.4.3开始，支持按照名称映射的方式

```xml
<constructor>
   <idArg column="id" javaType="int" name="id" />
   <arg column="age" javaType="_int" name="age" />
   <arg column="username" javaType="String" name="username" />
</constructor>
```

| 属性          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| `column`      | 数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 |
| `javaType`    | 一个 Java 类的完全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。 |
| `jdbcType`    | JDBC 类型，所支持的 JDBC 类型参见这个表格之前的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可能存在空值的列指定这个类型。 |
| `typeHandler` | 我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的完全限定名，或者是类型别名。 |
| `select`      | 用于加载复杂类型属性的映射语句的 ID，它会从 column 属性中指定的列检索数据，作为参数传递给此 select 语句。具体请参考关联元素。 |
| `resultMap`   | 结果映射的 ID，可以将嵌套的结果集映射到一个合适的对象树中。 它可以作为使用额外 select 语句的替代方案。它可以将多表连接操作的结果映射成一个单一的 `ResultSet`。这样的 `ResultSet` 将会将包含重复或部分数据重复的结果集。为了将结果集正确地映射到嵌套的对象树中，MyBatis 允许你 “串联”结果映射，以便解决嵌套结果集的问题。想了解更多内容，请参考下面的关联元素。 |
| `name`        | 构造方法形参的名字。从 3.4.3 版本开始，通过指定具体的参数名，你可以以任意顺序写入 arg 元素。参看上面的解释。 |

#### 关联

```xml
<association property="author" column="blog_author_id" javaType="Author">
	<id property="id" column="auhtor_id"/>
    <result propery="username" column="author_username"/>
</association>
```

关联的(association)元素处理 “有一个” 类型的关系，比如我们的示例中一个博客有一个用户，关联结果映射和其他类型的映射工作方式差不多，需要执行目标属性，以及属性的`javaType`

关联的不同之处是，告诉Mybatis如何加载关联。Mybatis有两种方式加载关联

* 嵌套select查询：通过执行另外一个SQL映射语句来加载期望的复杂类型
* 嵌套结果映射：使用嵌套的结果映射来处理连接结果的重复子集

普通的属性

| 属性          | 描述 |
| ------------- | ---- |
| `property`    |      |
| `javaType`    |      |
| `jdbcType`    |      |
| `typeHandler` |      |

##### 关联的嵌套 Select 查询 

<association>标签的属性

| 属性      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| column    | 数据库中的列名或别名。注意在使用复合主键时候，你可以通过`column={prop1=col1,prop2=col2}`这样的语法来指定多个传递给嵌套Select查询语句的别名，这会使得`prop1`和`prop2`作为参数对象，被设置为乔涛Select语句的参数 |
| select    | 用于加载复杂类型属性的映射语句ID，它会从column属性指定列中检索数据，作为参数传递给目标的select语句 |
| fetchType | 可选的。有效值为`lazy`和`eager`。指定属性后将在映射中忽略全局配置参数`lazyLoadingEnabled`，使用其属性 |

示例：

```xml
<resultMap id="blogResult" type="Blog">
	<association property="author" column="author_id" javaType="Author" select="selectAuthor"
</resultMap>
    <select id="selectBlog" resultMap="blogResult">
  SELECT * FROM BLOG WHERE ID = #{id}
</select>

<select id="selectAuthor" resultType="Author">
  SELECT * FROM AUTHOR WHERE ID = #{id}
</select>
```

两个select查询语句，一个用来加载博客（Blog）一个用来加载作者（Author），博客的结果映射描述了应该使用`selectAuthor`语句加载它的author属性。其他的属性会自动加载，只要他们的列名和属性名相匹配

这种方式会存在 N+1问题

##### 关联的嵌套结果映射

| 属性           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| `resultMap`    | 结果映射的ID，可以将此关联的嵌套结果映射到一个合适的对象树中。它可以将多表连接操作的结果映射为一个单一的`ResultSet`。这样的`ResultSet`有部分数据是重复的。为了将结果集正确的映射到嵌套的对象树种，MyBatis允许你“串联”结果映射 |
| `columnPrefix` | 当连接多个表时，你可能会不得不使用别名来避免在`ResultSet`中产生重复的列名，指定`columnPrefix`列名前缀允许你将带有这些前缀的列映射到一个外部的结果映射中 |
| notNullColumn  | 默认情况下，在至少一个被映射到的属性列不为空时，子对象才会被创建，你可以在这个属性上指定非空的列来改变默认行为，指定后Mybatis只在这些列任意一列非空时才创建一个子对象。 |
| autoMapping    | Mybatis将会为本结果映射开启或关闭自动映射                    |

嵌套结果例子：

```xml
<select id="selectBlog" resultMap="blogResult">
  select
    B.id            as blog_id,
    B.title         as blog_title,
    B.author_id     as blog_author_id,
    A.id            as author_id,
    A.username      as author_username,
    A.password      as author_password,
    A.email         as author_email,
    A.bio           as author_bio
  from Blog B left outer join Author A on B.author_id = A.id
  where B.id = #{id}
</select>
```



```xml
<resultMap id="blogResult" type="Blog">
  <id property="id" column="blog_id" />
  <result property="title" column="blog_title"/>
  <association property="author" column="blog_author_id" javaType="Author" resultMap="authorResult"/>
</resultMap>

<resultMap id="authorResult" type="Author">
  <id property="id" column="author_id"/>
  <result property="username" column="author_username"/>
  <result property="password" column="author_password"/>
  <result property="email" column="author_email"/>
  <result property="bio" column="author_bio"/>
</resultMap>
<!--如果不单独采用resultMap进行嵌套结果可以采用以下方式-->
<resultMap id="blogResult" type="Blog">
  <id property="id" column="blog_id" />
  <result property="title" column="blog_title"/>
  	<association property="author" column="blog_author_id" javaType="Author" >
      <id property="id" column="author_id"/>
      <result property="username" column="author_username"/>
      <result property="password" column="author_password"/>
      <result property="email" column="author_email"/>
      <result property="bio" column="author_bio"/>
    </association>
</resultMap>
```

>  在例子当中：id元素在嵌套结果映射中扮演非常重要的角色。你应该指定一个或多个唯一标识结果的属性，即使不指定这个属性,MyBatis仍然可以工作，但是会产生非常严重的性能问题，可以采用主键和联合主键的方式进行标识

假如一个博客关联两个作者，一作和二作，那么 Blog的ResultMap应该如下定义

```xml
<select id="selectBlog" resultMap="blogResult">
  select
    B.id            as blog_id,
    B.title         as blog_title,
    A.id            as author_id,
    A.username      as author_username,
    A.password      as author_password,
    A.email         as author_email,
    A.bio           as author_bio,
    CA.id           as co_author_id,
    CA.username     as co_author_username,
    CA.password     as co_author_password,
    CA.email        as co_author_email,
    CA.bio          as co_author_bio
  from Blog B
  left outer join Author A on B.author_id = A.id
  left outer join Author CA on B.co_author_id = CA.id
  where B.id = #{id}
</select>

<resultMap id="blogResult" type="Blog">
  <id property="id" column="blog_id" />
  <result property="title" column="blog_title"/>
  <association property="author"
    resultMap="authorResult" />
  <association property="coAuthor"
    resultMap="authorResult"
    columnPrefix="co_" />
</resultMap>
```

##### 关联的多结果集(ResultSet)

| 属性            | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| `column`        | 当使用多个结果集时，该属性指定结果集中用于与 `foreignColumn` 匹配的列（多个列名以逗号隔开），以识别关系中的父类型与子类型。 |
| `foreignColumn` | 指定外键对应的列名，指定的列将与父类型中 `column` 的给出的列进行匹配。 |
| `resultSet`     | 指定用于加载复杂类型的结果集名字。                           |

从版本 3.2.3 开始，MyBatis 提供了另一种解决 N+1 查询问题的方法。

某些数据库允许存储过程返回多个结果集，或一次性执行多个语句，每个语句返回一个结果集。 我们可以利用这个特性，在不使用连接的情况下，只访问数据库一次就能获得相关数据。

在例子中，存储过程执行下面的查询并返回两个结果集。第一个结果集会返回博客（Blog）的结果，第二个则返回作者（Author）的结果。

```xml
SELECT * FROM BLOG WHERE ID = #{id}

SELECT * FROM AUTHOR WHERE ID = #{id}
```

在映射语句中，必须通过 `resultSets` 属性为每个结果集指定一个名字，多个名字使用逗号隔开。

```xml
<select id="selectBlog" resultSets="blogs,authors" resultMap="blogResult" statementType="CALLABLE">
  {call getBlogsAndAuthors(#{id,jdbcType=INTEGER,mode=IN})}
</select>
```

现在我们可以指定使用 “authors” 结果集的数据来填充 “author” 关联：

```xml
<resultMap id="blogResult" type="Blog">
  <id property="id" column="id" />
  <result property="title" column="title"/>
  <association property="author" javaType="Author" resultSet="authors" column="author_id" foreignColumn="id">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="email" column="email"/>
    <result property="bio" column="bio"/>
  </association>
</resultMap>
```

你已经在上面看到了如何处理“有一个”类型的关联。但是该怎么处理“有很多个”类型的关联呢？这就是我们接下来要介绍的

#### 集合

```xml
<collection property="posts" ofType="domain.blog.Post">
  <id property="id" column="post_id"/>
  <result property="subject" column="post_subject"/>
  <result property="body" column="post_body"/>
</collection>
```

一个博客（Blog）只有一个作者（Author）。但是一个博客中有很多文章，在博客类中如下定义文章

```java
private List<Post> posts;
```

##### 集合的嵌套 Select 查询

```xml
<resultMap id="blogResult" type="Blog">
    <collection property="pots" javaType="ArrayList" column="id" ofType="Post" select="selectPostsForBlog"/>
</resultMap>

<select id = "selectPostsForBlog" resultType="Post">
    select * from post where BLOG_ID = #{id}
</select>

<select id="selectBlog" resultMap="blogResult">
	select * form blog where id = #{id}
</select>
```

##### 集合的嵌套结果映射

```xml
<select id="selectBlog" resultMap="blogResult">
  select
  B.id as blog_id,
  B.title as blog_title,
  B.author_id as blog_author_id,
  P.id as post_id,
  P.subject as post_subject,
  P.body as post_body,
  from Blog B
  left outer join Post P on B.id = P.blog_id
  where B.id = #{id}
</select>
```

```xml
<resultMap id="blogResult" type="Blog">
  <id property="id" column="blog_id" />
  <result property="title" column="blog_title"/>
  <collection property="posts" column="id" ofType="Post">
    <id property="id" column="post_id"/>
    <result property="subject" column="post_subject"/>
    <result property="body" column="post_body"/>
  </collection>
</resultMap>

<!--可重用的结果-->
```

##### 集合的多结果集

像关联元素那样，我们可以通过执行存储过程实现，它会执行两个查询并返回两个结果集，一个是博客的结果集，另一个是文章的结果集：

```
SELECT * FROM BLOG WHERE ID = #{id}

SELECT * FROM POST WHERE BLOG_ID = #{id}
```

在映射语句中，必须通过 `resultSets` 属性为每个结果集指定一个名字，多个名字使用逗号隔开。

```xml
<select id="selectBlog" resultSets="blogs,posts" resultMap="blogResult">
  {call getBlogsAndPosts(#{id,jdbcType=INTEGER,mode=IN})}
</select>
```

我们指定 “posts” 集合将会使用存储在 “posts” 结果集中的数据进行填充：

```xml
<resultMap id="blogResult" type="Blog">
  <id property="id" column="id" />
  <result property="title" column="title"/>
  <collection property="posts" ofType="Post" resultSet="posts" column="id" foreignColumn="blog_id">
    <id property="id" column="id"/>
    <result property="subject" column="subject"/>
    <result property="body" column="body"/>
  </collection>
</resultMap>
```

#### 鉴别器

```xml
<discriminator javaType="int" column="draft">
  <case value="1" resultType="DraftPost"/>
</discriminator>
```

有时候，一个数据库查询可能会返回多个不同的结果集（但总体上还是有一定的联系的）。 鉴别器（discriminator）元素就是被设计来应对这种情况的，另外也能处理其它情况，例如类的继承层次结构。 鉴别器的概念很好理解——它很像 Java 语言中的 switch 语句。

一个鉴别器的定义需要指定 column 和 javaType 属性。column 指定了 MyBatis 查询被比较值的地方。 而 javaType 用来确保使用正确的相等测试（虽然很多情况下字符串的相等测试都可以工作）。例如：

```xml
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

在这个示例中，MyBatis 会从结果集中得到每条记录，然后比较它的 vehicle type 值。 如果它匹配任意一个鉴别器的 case，就会使用这个 case 指定的结果映射。 这个过程是互斥的，也就是说，剩余的结果映射将被忽略（除非它是扩展的，我们将在稍后讨论它）。 如果不能匹配任何一个 case，MyBatis 就只会使用鉴别器块外定义的结果映射。 所以，如果 carResult 的声明如下：

```xml
<resultMap id="carResult" type="Car">
  <result property="doorCount" column="door_count" />
</resultMap>
```

那么只有 doorCount 属性会被加载。这是为了即使鉴别器的 case 之间都能分为完全独立的一组，尽管和父结果映射可能没有什么关系。在上面的例子中，我们当然知道 cars 和 vehicles 之间有关系，也就是 Car 是一个 Vehicle。因此，我们希望剩余的属性也能被加载。而这只需要一个小修改。

```xml
<resultMap id="carResult" type="Car" extends="vehicleResult">
  <result property="doorCount" column="door_count" />
</resultMap>
```

现在 vehicleResult 和 carResult 的属性都会被加载了。

可能有人又会觉得映射的外部定义有点太冗长了。 因此，对于那些更喜欢简洁的映射风格的人来说，还有另一种语法可以选择。例如：

```xml
<resultMap id="vehicleResult" type="Vehicle">
  <id property="id" column="id" />
  <result property="vin" column="vin"/>
  <result property="year" column="year"/>
  <result property="make" column="make"/>
  <result property="model" column="model"/>
  <result property="color" column="color"/>
  <discriminator javaType="int" column="vehicle_type">
    <case value="1" resultType="carResult">
      <result property="doorCount" column="door_count" />
    </case>
    <case value="2" resultType="truckResult">
      <result property="boxSize" column="box_size" />
      <result property="extendedCab" column="extended_cab" />
    </case>
    <case value="3" resultType="vanResult">
      <result property="powerSlidingDoor" column="power_sliding_door" />
    </case>
    <case value="4" resultType="suvResult">
      <result property="allWheelDrive" column="all_wheel_drive" />
    </case>
  </discriminator>
</resultMap>
```

#### 自动映射

在简单的场景下，Mybatis可以为你自动映射查询结果，但是如果遇到复杂的场景，你需要构建一个结果映射。但是Mybatsi可以混合使用这两种场景。

自动映射查询结果时，Mybatis会获取结果中返回的列名，并在Java类中查找相同名字的属性（忽略大小写）。

通常数据库使用大写字母组成单词名称，单词间用下划线分隔；而Java属性一般遵循驼峰命名。为了在两种命名方式之间启用自动映射，需要将`mapUnderscoreToCamelCase`设置为true

MyBatis有三种自动映射等级

* `NONE`-禁用自动映射。
* `PARTIAL`-对除在内部定义了嵌套结果映射以外的属性进行映射
* `FULL`-自动映射所有属性

默认值是 `PARTIAL`，这是有原因的。当对连接查询的结果使用 `FULL` 时，连接查询会在同一行中获取多个不同实体的数据，因此可能导致非预期的映射。 下面的例子将展示这种风险：

```xml
<select id="selectBlog" resultMap="blogResult">
  select
    B.id,
    B.title,
    A.username,
  from Blog B left outer join Author A on B.author_id = A.id
  where B.id = #{id}
</select>
<resultMap id="blogResult" type="Blog">
  <association property="author" resultMap="authorResult"/>
</resultMap>

<resultMap id="authorResult" type="Author">
  <result property="username" column="author_username"/>
</resultMap>
```

在该结果映射中，*Blog* 和 *Author* 均将被自动映射。但是注意 *Author* 有一个 *id* 属性，在 ResultSet 中也有一个名为 *id* 的列，所以 Author 的 id 将填入 Blog 的 id，这可不是你期望的行为。 所以，要谨慎使用 `FULL`。

无论设置的自动映射等级是哪种，你都可以通过在结果映射上设置 `autoMapping` 属性来为指定的结果映射设置启用/禁用自动映射。

```xml
<resultMap id="userResultMap" type="User" autoMapping="false">
  <result property="password" column="hashed_password"/>
</resultMap>
```

> Configuration中 一些默认设置
>
> * `autoMappingBehavior` -默认值为 PARTIAL
> * `mapUnderscoreToCamelCase`-默认值为false
> * `autoMappingUnknownColumnBehavior`-默认值为false

#### 缓存

默认情况下，只启用本地的会话缓存，它仅仅在一个会话中的数据进行缓存。要启用全局的二级缓存，只需要在SQL映射文件中添加一行

```xml
<cache/>
```

这个简单的语句效果如下：

* 映射语句文件中所有 select 语句的结果将会被缓存
* 映射语句文件中所有的 insert、update和delete语句会刷新缓存
* 缓存会使用最近最少使用算法（LRU,Least Recently Used）
* 缓存会保存列表或对象1024个引用
* 缓存不会定时刷新
* 缓存会被视为读/写缓存，这意味着获取到的对象并不是共享的，可以安全的被调用者修改

提示：缓存只作用于 cache 标签所在的映射文件中的语句，即通常说的 namespace 范围，如果混合使用Java API和 XML映射文件，在共用接口中的语句不会被默认缓存，需要使用@CacheNamespaceRef注解指定缓存作用域

修改缓存的属性

```xml
<cache 
       <!--默认值为LRU-->
       eviction="FIFO"
       <!--以毫秒为单位的合理值-->
       flushInterval="60000"
	   <!--默认值为1024-->
       size="512"
	   <!--默认值为true-->
       readOnly="false"/>
```

缓存清楚策略：

* `LRU`-最近最少使用：移除最长时间不被使用的对象
* `FIFO`-先进先出：按对象进入缓存的顺序来移除他们
* `SOFT`-软引用：基于垃圾回收器状态和软引用规则移除对象
* `WEAK`-弱引用：更积极的基于垃圾收集器状态和弱引用规则移除对象

> 二级缓存是有事务性的。这意味着，当SqlSession完成并提交时，或是完成并回滚时，但没有执行flushCache=true的insert/update/delete语句时，缓存会获得更新

##### 自定义缓存

```java
// 自定义缓存接口
public interface Cache {
  String getId();
  int getSize();
  void putObject(Object key, Object value);
  Object getObject(Object key);
  boolean hasKey(Object key);
  Object removeObject(Object key);
  void clear();
}
```

为了对你的缓存进行配置，只需要简单地在你的缓存实现中添加公有的 JavaBean 属性，然后通过 cache 元素传递属性值，例如，下面的例子将在你的缓存实现上调用一个名为 `setCacheFile(String file)` 的方法：

```xml
<cache type="com.domain.something.MyCustomCache">
  <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
</cache>
```

你可以使用所有简单类型作为 JavaBean 属性的类型，MyBatis 会进行转换。 你也可以使用占位符（如 `${cache.file}`），以便替换成在[配置文件属性](https://mybatis.org/mybatis-3/zh/configuration.html#properties)中定义的值。

从版本 3.4.2 开始，MyBatis 已经支持在所有属性设置完毕之后，调用一个初始化方法。 如果想要使用这个特性，请在你的自定义缓存类里实现 `org.apache.ibatis.builder.InitializingObject` 接口。

```java
public interface InitializingObject {
  void initialize() throws Exception;
}
```

##### cache-ref

回想一下上一节的内容，对某一命名空间的语句，只会使用该命名空间的缓存进行缓存或刷新。 但你可能会想要在多个命名空间中共享相同的缓存配置和实例。要实现这种需求，你可以使用 cache-ref 元素来引用另一个缓存。

```xml
<cache-ref namespace="com.someone.application.data.SomeMapper"/>
```
