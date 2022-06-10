# 自定义持久层框架

### DBC问题分析

1. 数据库配置信息存在硬编码问题（配置文件）
2. 频繁创建释放数据库连接（连接池）
3. sql语句、设置参数，获取结果集参数均存在硬编码问题（配置文件）
4. 需要手动封装返回结果集，较为繁琐（反射，内省自动封装）

### 自定义持久层框架设计思路

* 使用端：（项目）引入自定义持久层框架的Jar包

  > 提供两部分配置信息：数据库配置信息、sql配置信息
  >
  > 使用配置文件提供这两部分配置信息
  >
  > 1. sqlMapConfig.xml：存放数据库配置信息，存放mapper.xml的全路径
  > 2. mapper.xml：存放sql配置信息

* 自定义持久层框架本身：（工程）本质就是对JDBC进行了封装

  > 1. 加载配置文件：根据配置文件路径，加载配置文件为字节输入流，存储在内存中
  >
  >    创建Resource类 方法：InputStream getResourceAsStream(String path)
  >
  > 2. 创建两个javaBean：（容器对象）存放的是对配置文件解析出来的内容
  >
  >    Configuratin：核心配置类，存放sqlMapper.cml解析出来的内容
  >
  >    MappedStatemant：映射配置类，存放mapper.xml解析出来的内容
  >
  > 3. 解析配置文件：dom4j
  >
  >    创建类：SqlSessionFactoryBuilder 方法： build（InputStream in）
  >
  >    第一：使用dom4j解析配置文件，将解析出来的内容封装发哦容器对象中
  >
  >    第二：创建SqlSessionFactory对象(生产SqlSession会话对象)
  >
  > 4. 创建SqlSessionFactory接口及实现类DefaultSqlSessionFactory
  >
  >    第一：openSession()：生产sqlSession
  >
  > 5. 创建SqlSession接口及其实现类 DefaultSqlSession
  >
  >      定义对数据库的crud操作：
  >
  >      selectList()
  >
  >      selectOne()
  >
  >      update()
  >
  >      delete()
  >
  > 6. 创建Executor接口及其实现类SimpleExecutor实现类
  >
  >     query(Configuration,MappedStatement,Object...params)：执行的就是JDBC代码

### 自定义框架实现

 

# Mybatis相关概念

Mybatis是一款优秀的基于ORM的半自动轻量级持久层框架

