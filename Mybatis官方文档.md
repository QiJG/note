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

# 配置

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

| 设置名             | <span style="display:inline-block;width: 70px"> 描述</span>  | 有效值      | <span style="display:inline-block;width: 80px"> 默认值</span> | <span style="display:inline-block;width: 120px">疑惑</span>  |
| ------------------ | ------------------------------------------------------------ | ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| cacheEnabled       | 全局性打开或关闭所有映射器配置文件中已配置的任何缓存         | true\|false | true                                                         | 如果是true则开启二级缓存（使用CacheExecitor），且当mapper中配置了<cache>标签才会生效（添加缓存） |
| lazyLoadingEnabled | 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。特定关联关系中可以通过设置fetchType属性覆盖该项开关状态 | true\|false | false                                                        |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |
|                    |                                                              |             |                                                              |                                                              |

