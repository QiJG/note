# SpringMVC 框架

## 第一部分 Servlet介绍

 ###  1.Servlet简介

Java Servlet 是运行在Web服务器或应用服务器上的程序，它是作为Web浏览器或其他Http客户端的请求和Http服务器上的数据库或应用程序之间的中间层

### 2.Servlet 任务

* 读取客户端发送的数据
* 处理数据并生成结果
* 发送响应到客户端

### 3.Servlet 架构

![image-20220917180524903](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220917180524903.png)

Servlet体系

![image-20220918164708394](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220918164708394.png)

###  4.Servlet生命周期

* Servlet初始化后调用init方法
* Servlet调用service()方法来处理客户端请求
* Servlet销毁前调用destroy()方法
* 最后Servlet由JVM的垃圾回收器进行垃圾回收

   Servlet的入口：

> Servlet实际上是tomcat容器生成的，调用init方法可以初始化。他有别于普通java的执行过程，普通java需要main方法；但是web项目由于是服务器控制的创建和销毁，所以servlet的访问也需要tomcat来控制。通过tomcat访问servlet的机制是通过使用http协议的URL来访问，所以servlet的配置完想要访问，必须通过URL来访问，所以没有main方法。

### 5.转发与重定向

![image-20220920224938356](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220920224938356.png)

![image-20220920224956291](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220920224956291.png)

![image-20220920225054863](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220920225054863.png)



## 第二部分 SpringMVC基础篇

### 1.SpringMVC介绍

#### 1.SpringMVC请求处理流程

![image-20221009182703718](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221009182703718.png)

#### 2.SpringMVC九大组件

* HandlerMapping（处理器映射器）

  HandlerMapping 是⽤来查找 Handler 的，也就是处理器，具体的表现形式可以是类，也可以是

  ⽅法。⽐如，标注了@RequestMapping的每个⽅法都可以看成是⼀个Handler。Handler负责具

  体实际的请求处理，在请求到达后，HandlerMapping 的作⽤便是找到请求相应的处理器Handler 和 Interceptor.

* HandlerAdapter（处理器适配器）

  HandlerAdapter 是⼀个适配器。因为 Spring MVC 中 Handler 可以是任意形式的，只要能处理请

  求即可。但是把请求交给 Servlet 的时候，由于 Servlet 的⽅法结构都是

  doService(HttpServletRequest req,HttpServletResponse resp)形式的，要让固定的 Servlet 处理

  ⽅法调⽤ Handler 来进⾏处理，便是 HandlerAdapter 的职责。

* HandlerExceptionResolver

  HandlerExceptionResolver ⽤于处理 Handler 产⽣的异常情况。它的作⽤是根据异常设置

  ModelAndView，之后交给渲染⽅法进⾏渲染，渲染⽅法会将 ModelAndView 渲染成⻚⾯。

* ViewResolver

  ViewResolver即视图解析器，⽤于将String类型的视图名和Locale解析为View类型的视图，只有⼀

  个resolveViewName()⽅法。从⽅法的定义可以看出，Controller层返回的String类型视图名

  viewName 最终会在这⾥被解析成为View。View是⽤来渲染⻚⾯的，也就是说，它会将程序返回

  的参数和数据填⼊模板中，⽣成html⽂件。ViewResolver 在这个过程主要完成两件事情：

  ViewResolver 找到渲染所⽤的模板（第⼀件⼤事）和所⽤的技术（第⼆件⼤事，其实也就是找到

  视图的类型，如JSP）并填⼊参数。默认情况下，Spring MVC会⾃动为我们配置⼀个

  InternalResourceViewResolver,是针对 JSP 类型视图的。

* RequestToViewNameTranslator

  RequestToViewNameTranslator 组件的作⽤是从请求中获取 ViewName.因为 ViewResolver 根据

  ViewName 查找 View，但有的 Handler 处理完成之后,没有设置 View，也没有设置 ViewName，

  便要通过这个组件从请求中查找 ViewName。

* LocaleResolver

* ViewResolver 

  组件的 resolveViewName ⽅法需要两个参数，⼀个是视图名，⼀个是 Locale。

  LocaleResolver ⽤于从请求中解析出 Locale，⽐如中国 Locale 是 zh-CN，⽤来表示⼀个区域。这

  个组件也是 i18n 的基础。

* ThemeResolver

  ThemeResolver 组件是⽤来解析主题的。主题是样式、图⽚及它们所形成的显示效果的集合。

  Spring MVC 中⼀套主题对应⼀个 properties⽂件，⾥⾯存放着与当前主题相关的所有资源，如图

  ⽚、CSS样式等。创建主题⾮常简单，只需准备好资源，然后新建⼀个“主题名.properties”并将资

  源设置进去，放在classpath下，之后便可以在⻚⾯中使⽤了。SpringMVC中与主题相关的类有

  ThemeResolver、ThemeSource和Theme。ThemeResolver负责从请求中解析出主题名，

  ThemeSource根据主题名找到具体的主题，其抽象也就是Theme，可以通过Theme来获取主题和

  具体的资源。

* MultipartResolver

  MultipartResolver ⽤于上传请求，通过将普通的请求包装成 MultipartHttpServletRequest 来实

  现。MultipartHttpServletRequest 可以通过 getFile() ⽅法 直接获得⽂件。如果上传多个⽂件，还

  可以调⽤ getFileMap()⽅法得到Map<FileName，File>这样的结构，MultipartResolver 的作⽤就

  是封装普通的请求，使其拥有⽂件上传的功能。

* FlashMapManager

  FlashMap ⽤于重定向时的参数传递，⽐如在处理⽤户订单时候，为了避免重复提交，可以处理完

  post请求之后重定向到⼀个get请求，这个get请求可以⽤来显示订单详情之类的信息。这样做虽然

  可以规避⽤户重新提交订单的问题，但是在这个⻚⾯上要显示订单的信息，这些数据从哪⾥来获得

  呢？因为重定向时么有传递参数这⼀功能的，如果不想把参数写进URL（不推荐），那么就可以通

  过FlashMap来传递。只需要在重定向之前将要传递的数据写⼊请求（可以通过ServletRequestAttributes.getRequest()⽅法获得）的属性OUTPUT_FLASH_MAP_ATTRIBUTE

  中，这样在重定向之后的Handler中Spring就会⾃动将其设置到Model中，在显示订单信息的⻚⾯

  上就可以直接从Model中获取数据。FlashMapManager 就是⽤来管理 FalshMap 的。

### 2.SpringMVC项目搭建

#### 1.POM依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cpm.custom</groupId>
    <artifactId>spring-ssm</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <name>spring-ssm Maven Webapp</name>
    <url>http://www.example.com</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <spring.version>5.1.12.RELEASE</spring.version>
    </properties>

    <dependencies>
        <!-- 持久层依赖 开始 -->
        <!-- spring ioc组件需要的依赖包 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- spring 事务管理和JDBC依赖包 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- mysql数据库驱动包 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.49</version>
        </dependency>

        <!-- dbcp连接池的依赖包 -->
        <dependency>
            <groupId>commons-dbcp</groupId>
            <artifactId>commons-dbcp</artifactId>
            <version>1.4</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.21</version>
        </dependency>

        <!-- mybatis依赖 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.4.5</version>
        </dependency>

        <!-- mybatis和spring的整合依赖 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.3.1</version>
        </dependency>

        <!-- 持久层依赖 结束 -->

        <!-- 业务层依赖 开始 -->
        <!-- 基于AspectJ的aop依赖 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>aopalliance</groupId>
            <artifactId>aopalliance</artifactId>
            <version>1.0</version>
        </dependency>
        <!-- 业务层依赖 结束 -->


        <!-- 表现层依赖 开始 -->
        <!-- spring MVC依赖包 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- jstl 取决于视图对象是否是JSP -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>jstl</artifactId>
            <version>1.2</version>
        </dependency>

        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>


        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-oxm</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>com.thoughtworks.xstream</groupId>
            <artifactId>xstream</artifactId>
            <version>1.4.10</version>
        </dependency>


        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.6</version>
        </dependency>


        <!-- 表现层依赖 结束 -->

        <!-- spring 单元测试组件包 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- 单元测试Junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>

        <!-- Mock测试使用的json-path依赖 -->
        <dependency>
            <groupId>com.jayway.jsonpath</groupId>
            <artifactId>json-path</artifactId>
            <version>2.2.0</version>
        </dependency>


        <!-- 文件上传 -->
        <dependency>
            <groupId>commons-fileupload</groupId>
            <artifactId>commons-fileupload</artifactId>
            <version>1.3.1</version>
        </dependency>

        <dependency>
            <groupId>com.itfsw</groupId>
            <artifactId>mybatis-generator-plugin</artifactId>
            <version>1.3.9</version>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.17.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.17.2</version>
        </dependency>


    </dependencies>

    <build>
        <finalName>spring-ssm</finalName>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                    <include>**/*.properties</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>
        <plugins>
            <!--配置maven编译级别-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <!-- 代码生成器插件-->
            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.3.7</version>
                <executions>
                    <execution>
                        <id>Generate MyBatis Artifacts</id>
                        <!--解决每次编译或打包时都会重新执行编译插件-->
                        <phase>deploy</phase>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <!-- 配置文件 -->
                    <configurationFile>src/main/resources/generatorConfig.xml</configurationFile>
                    <!-- 允许移动和修改 -->
                    <verbose>true</verbose>
                    <overwrite>true</overwrite>
                </configuration>
                <dependencies>
                    <!-- jdbc 依赖 -->
                    <dependency>
                        <groupId>mysql</groupId>
                        <artifactId>mysql-connector-java</artifactId>
                        <version>5.1.49</version>
                    </dependency>
                    <dependency>
                        <groupId>com.itfsw</groupId>
                        <artifactId>mybatis-generator-plugin</artifactId>
                        <version>1.3.9</version>
                    </dependency>
                </dependencies>
            </plugin>
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <path>/</path>
                    <port>8080</port>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

```

#### 2.web.xml的配置

```xml
 <!--1.web.xml中 context-param listener filter servelet 加载顺序-->
  <!-- context-param listener  filter  servlet-->
  <!--2.load-on-startup 标签的作用-->
  <!-- 表示servlet的加载顺序 如果是正数则 tomcat启动时就加载，数字越小 优先级越高，如果为负数则表示使用时才会加载-->
  <!--3.url-pattern标签的配置方式：/dispatcherServlet /servlet/* /* / 优先级是如何的-->
  <!--匹配的优先级为 精确匹配 /* *.do /-->
  <!--4.url-pattern 标签配置 /* 报错，原因是它拦截了JSP请求，但是又不能处理JSP请求
  为什么配置/就不拦截JSP请求配置/*就会拦截JSP请求 -->
  <!--因为tomcat默认有一个 web.xml 内部配置了两个servlet DefaultServlet JspServlet
  -->

  <!--配置spring mvc去读取spring配置文件之后 就产生了父子容器的问题-->
  <!-- 在servlet中配置的springmvc文件为子容器 context-param 中配置的是父容器，子容器可以注入使用父容器的对象-->
  <!--配置前端控制器-->
  <servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <load-on-startup>2</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <!-- 不要配置为 /* 否则会报错-->
    <!--/* 会拦截整个项目的资源访问 包含jsp和静态资源的访问，对于静态资源的访问 springMVC提供了默认的Handler处理器-->
    <url-pattern>/</url-pattern>
  </servlet-mapping>
  <context-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring/application-*.xml</param-value>
  </context-param>
  <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
   </listener>
```

#### 3.spring-mvc配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <!--扫描controller-->
    <context:component-scan base-package="com.custom.controller,com.custom.advice"/>

    <!--配置处理器映射器和处理器适配器-->
    <mvc:annotation-driven conversion-service="conversionService"/>

    <!--配置视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/"></property>
        <property name="suffix" value=".jsp"></property>
    </bean>
    <!--转换器配置-->
    <bean id="conversionService"
          class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <bean class="com.custom.converter.DateConverter"> </bean>
            </set>
        </property>
    </bean>
    <!--静态资源的处理-->
    <mvc:default-servlet-handler/>
    <mvc:interceptors>
        <!--全局拦截器-->
        <bean class="com.custom.interceptor.MyInterceptor01"></bean>
        <mvc:interceptor>
            <mvc:mapping path="/handle01"/>
            <bean class="com.custom.interceptor.MyInterceptor02"></bean>
        </mvc:interceptor>
    </mvc:interceptors>
    <!--配置multipart解析器-->
    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <!--最大上传尺寸为5M-->
        <property name="maxUploadSize" value="524880"></property>
    </bean>

    <bean class="com.custom.handler.RunTimeExceptionResolver"/>

    <!--处理器的配置-->
    <bean class="com.custom.handler.DemoHandler"/>
    <!--配置处理器映射器-->
    <bean name="/helloHandler" class="com.custom.handler.DemoHandler"/>
    <!--配置 http请求处理器适配器-->
    <bean class="com.custom.handler.HttpRequestHandlerAdapter"/>
</beans>
```

#### 4.application-dao.xml配置

````xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <!--加载db.properties-->
    <context:property-placeholder location="classpath:db.properties"></context:property-placeholder>

    <!--配置数据源-->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${jdbc.driverClassName}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="username" value="${jdbc.username}"> </property>
        <property name="password" value="${jdbc.password}"></property>
        <property name="testOnBorrow" value="true"/>
        <property name="maxActive" value="100"/>
        <property name="minIdle" value="10"/>
        <property name="validationQuery" value="select 1 from dual"/>
    </bean>

    <!--配置sqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="typeAliasesPackage" value="com.custom.mapper"/>
    </bean>
    <!--配置mapper扫描器-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.custom.mapper"></property>
    </bean>
</beans>
````

#### 5.application-service.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
           http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd
">
    <!--扫描bean-->
    <context:component-scan base-package="com.custom.service"/>

    <!--事务管理器-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!--事务-->
    <tx:advice id="interceptor" transaction-manager="transactionManager" >
        <tx:attributes>
            <tx:method name="query*" propagation="SUPPORTS"/>
            <tx:method name="add*" propagation="REQUIRED"/>

        </tx:attributes>
    </tx:advice>
    <aop:config>
        <aop:advisor advice-ref="interceptor" pointcut="execution(* *..*.*Impl.*(..))"/>
    </aop:config>

</beans>
```

#### 6.编写Controller类

``` java
@Controller
public class DemoController {
    /**
     * 返回值的设置
     * 不添加注解的情况下
     * 返回 ModelAndView
     * 返回 void
     * 返回 String
     */

    @RequestMapping(value = "/handle00")
    @ResponseBody
    public String handle00(String id, String name, Model model) {
        System.out.println("(handl00)id:" + id + ",name:" + name);
        if (model != null) {
            Map<String, Object> modelMap = model.asMap();
            System.out.println(modelMap);
        }
        return "sucess";
    }
}
```

### 3.SpringMVC高级属性

#### 1.返回值的类型

* ModelAndView

  > Controller方法定义ModelAndView对象并返回，在对象中增加model数据，指定view

* void

  > 请求转发：
  >
  > ```java
  > request.getRequestDispatcher("/handle00").forward(request, response);
  > ```
  >
  > 重定向:
  >
  > ```java
  > response.sendRedirect("/handle00?id=1");
  > ```
  >
  > 直接返回响应数据：
  >
  > ```java
  > response.setCharacterEncoding("UTF-8");
  > response.setContentType("application/json;charset=utf-8");
  > response.getWriter().write("张三shenqi");
  > ```

* String

  > String（推荐）
  >
  > 逻辑视图名：
  >
  > ```
  > return "success";
  > ```
  >
  > 转发：
  >
  > ```
  > return "forward:/handle00";
  > ```
  >
  > 重定向：
  >
  > ```
  > return "redirect:/handle00";
  > ```
  >
  > @ResponseBody注解
  >
  > 如果返回字符串则返回字符串，如果返回对象则会序列化为json

#### 2.请求参数的类型

内置支持：

* HttpServletRequest

* HttpServletResponse
* HttpSession
* InputStream,OutputStream
* Reader,writer
* Model/ModelMap(BingAwareModelMap)

简单参数绑定

* 直接绑定

  > http请求参数的[Key]和controller方法的形参名称一致

* 注解绑定

  > 采用@RequestParam注解进行绑定
  >
  > value,required,defaultValue

* 绑定POJO类型

  > 要求表单中[参数名称]和controller方法中的[pojo形参属性名称]保持一致

* 绑定集合或数组类型

  简单类型数据通过 String[]或者 pojo[] 属性接收，不能用List集合接收

  批量传递的请求参数如果最终使用List<POJO>接收，那么这个属性必须在另一个POJO类中

#### 3.@RequestMapping属性

​	@RequestMapping(value,method,params)

## 第三部分 SpringMVC源码解析

源码解析的初始化入口为：

```java
// DispatcherServlet
@Override
	protected void onRefresh(ApplicationContext context) {
		// 调用初始化策略
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		// 初始化9个组件
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```

源码解析调用入口为：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 检查是不是上传请求
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 根据request查询到Handler
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				// 根据handler找到对应的HandlerAdapter
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				// 处理GET HEAD请求的LastModified
				// 当浏览器第一次根服务器请求资源(GET HEAD 请求时) 服务器在返回的请求头里会包含一个Last-Modified的属性
				// 代表本资源最后时什么时候修改的，在浏览器以后发送请求时会同时发送之前接收到的Lat-Modified，服务器接收到Last-Modified的请求后
				//会用其值和自己实际资源最后修改时间做对比，如果资源过期了 则返回心得资源同时返回新的Last-modified否则直接返回304表示资源未过期
				//浏览器直接使用之前缓存的结果
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
				// 执行响应的拦截器 Interceptor.preHandle
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 用HandlerAdapter处理Handler
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				// 如果需要异步处理直接返回
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
				// 当view为空时 比如handler返回值为void 根据request设置默认的view
				applyDefaultViewName(processedRequest, mv);
				// 执行响应的Interceptor的postHandle
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 调用processDispatchResult方法处理上面的结果，包含找到view并渲染输出给用户
			// 处理返回结果 包括处理异常 渲染页面 发出完成通知触发Interceptor.afterCompletion
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			// 判断是否是异步请求
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				// 删除上传请求的资源
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```



