---
layout:     post
title:      Springmvc mybatis java配置整合
subtitle:   ''
date:       2018-04-18
author:     Logic
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Springmvc mybatis java配置整合
    - spring mvc
    - mybatis
    - java config
---

​    从Servlet3.0以后，在发布web项目的时候都已经支持Java的配置了，本文将Spring mvc、Mybatis进行整个，并利用Maven的方式将进行构建，采取`Java`的方式进行配置，做到除了`pom.xml`以外，做到项目的零xml的配置。

### 项目目录结构

```
${basedir}
├── pom.xml
├── README.md
├── springmvcjava.iml
└── src
    ├── main
    │   ├── filters
    │   │   ├── beta
    │   │   │   └── jdbc.properties
    │   │   ├── dev
    │   │   │   └── jdbc.properties
    │   │   ├── local
    │   │   │   ├── jdbc.properties
    │   │   │   └── logback.xml
    │   │   └── prod
    │   │       └── jdbc.properties
    │   ├── java
    │   │   └── com
    │   │       └── logic
    │   │           ├── config
    │   │           │   ├── DataConfig.java
    │   │           │   ├── RootConfig.java
    │   │           │   ├── WebAppInitializer.java
    │   │           │   └── WebConfig.java
    │   │           ├── controller
    │   │           │   └── UserController.java
    │   │           ├── dao
    │   │           │   └── UserDao.java
    │   │           ├── model
    │   │           │   └── User.java
    │   │           ├── service
    │   │           │   ├── impl
    │   │           │   │   └── UserServiceImpl.java
    │   │           │   └── UserService.java
    │   │           └── util
    │   │               └── Utils.java
    │   ├── resources
    │   │   └── mapper
    │   │       └── UserDao.xml
    │   └── webapp
    │       ├── index.jsp
    │       └── WEB-INF
    └── test
        ├── filters
        │   ├── beta
        │   │   └── jdbc.properties
        │   ├── dev
        │   │   └── jdbc.properties
        │   ├── local
        │   │   ├── jdbc.properties
        │   │   └── logback.xml
        │   └── prod
        │       └── jdbc.properties
        ├── java
        │   └── com
        │       └── logic
        │           └── dao
        │               └── UserDaoTest.java
        └── resources
```

该目录结构为标准的maven web项目目录结构。

- `/src/main/filters`目录下面放了针对不同的环境所需要过滤的资源，本例中使用对不同环境下链接数据库的不同进行过滤。
- `/src/main/java`为项目的源码目录。
  - 在该目录中，model、dao、controller、service包与使用xml配置时一样的，没有太大变化
  - `config`目录是为了使用java的配置新建的，在下面会主要讲述各个文件的作用以及内容。
- `/src/main/resources`为项目资源目录，本例中除了mybatis的Mapper文件外没有设置任何资源，在进行构建的时候，`/src/main/filters`目录下的某个环境中的资源文件会被构建到该资源文件中。
- `/src/test`下的目录结构和`/src/main/` 下的结构差不多，`/src/test/filters`存放的是与项目测试相关的过滤资源，`/src/test/java`中为测试源码，`/src/main/resources` 中存放的是测试用资源。

### `config`包中各个文件的内容及作用

​    此包中文件的主要作用是对Spring 应用上下文、Spring mvc应用上下文、web以及mybatis进行配置。其中`RootConfig.java`文件的作用是配置spring配置应用上下文，就相当于使用xml进行配置时的`applicationContext.xml`。`WebConfig.java`文件的作用是配置spring mvc应用上下文，就相当于使用xml进行配置时的`spring-mvc.xml`，主要对`DispatcherServlet`进行配置。WebAppInitializer.java可以想象成使用xml进行配置时的`web.xml`，此文件对Servlet进行配置。`DataConfig.java`就相当于使用xml配置mybatis时的`mybatis.xml`。既然都能找到与xml配置相类似的文件，这样就好理解了。下面针对每个不同的文件进行详细的阐述。
#### `RootConfig.java`

```java
/**
 * 配置Spring应用上下文相关的Bean
 *
 * @author jiming.luo
 * @date 2018/1/29 20:59
 * @since 1.0
 */
@Configuration
@ComponentScan(basePackages = "com.logic", excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)})
@Import(com.logic.config.DataConfig.class)
public class RootConfig {
}
```

在本类中没有任何内容，只是添加了相关的注解，如果想配置一些第三方的bean,直接在该勒种进行配置即可，

- `@Configuration` 注解说明本文件是个配置文件。
- `@ComponentScan `注解相当于xml中的`<context:component-scan />`标签， 开启注解扫描。
- `@Import` 注解相当于xml中的`<import /> `标签，引入了外部的配置类，如果一些插件或者应用现在还不支持使用Java进行配置，可以使用xml的方式进行配置，然后利用`@ImportResource`注解将xml配置文件引入进来

##### `WebConfig.java`

```java
/**
 * 对Spring mvc进行配置
 *
 * @author jiming.luo
 * @date 2018/1/29 20:07
 * @since 1.0
 */
@Configuration
@EnableWebMvc
@ComponentScan("com.logic.controller")
public class WebConfig extends WebMvcConfigurerAdapter {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        /*配置视图解析器*/
        viewResolver.setPrefix("/WEB-INF/views");
        viewResolver.setPrefix("jsp");
        return viewResolver;
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        /*配置静态资源*/
        configurer.enable();
    }
}
```

该文件对`DispatcherServlet`进行配置，利用`@Bean`注解进行bean的声明，就相当于xml配置中的`<bean> `标签，在本类中配置了jsp视图解析器`ViewResolver`，采用`InternalResourceViewResolver`对其进行创建，并对其进行了相关的配置。并配置相关的静态资源。在本类中可以找到与配置`springmvc.xml`相同的配置方式。

##### `WebAppInitializer.java`

```java
/**
 * 在容器启动的时候，会加载会利用此类进行Servlet的配置
 *
 * @author jiming.luo
 * @date 2018/1/29 20:53
 * @since 1.0
 */
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    /**
     * 配置应用上下文
     *
     * @return
     */
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    /**
     * 配置spring web上下文
     *
     * @return
     */
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    /**
     * 配置映射关系
     *
     * @return
     */
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
        characterEncodingFilter.setEncoding("UTF-8");
        characterEncodingFilter.setForceEncoding(true);
        return new Filter[]{characterEncodingFilter};
    }
}
```

在理解此类的时候，如前面所述，可以将其想象成`web.xml`就行了，`web.xml`中的配置都会此类中找到相对应的配置方法。

> 在web 3.0环境中，容器会在类路径中查找实现`javax.servlet.ServletContainerInitializer`接口的类，如果能发现的话，就利用其配置`Servlet`容器。`Spring`提供了这个接口的实现，名为`SpringServletContainerInitializer`，这个类又会反过来查找实现`WebApplicationInitializer`的类，并将配置的任务交给他们来完成，`AbstractAnnotationConfigDispatcherServletInitializer`就是`WebApplicationInitializer`类的实现。

- 方法`protected Class<?>[] getRootConfigClasses()` 加载应用的上下文进行配置，在其中可以添加多个应用上下文，将其所对应的类加载进来即可。
- 方法`protected Class<?>[] getServletConfigClasses()` 加载对DispatcherServlet的配置。
- 方法`protected String[] getServletMappings()` 相当于在`web.xml`文件中配置了`DispatcherServlet`所对应的`servlet-mapping`，默认会处理所有的请求。
- 方法`protected Filter[] getServletFilters()` 对编码进行进行配置。

##### `DataConfig.java`

````java
/**
 * Mybatis 相关配置
 *
 * @author jiming.luo
 * @date 2018/1/30 10:25
 * @since 1.0
 */
@Configuration
@MapperScan(basePackages = "com.logic.dao")
@PropertySource(value = "classpath:jdbc.properties")
public class DataConfig {

    @Resource
    private Environment environment;

    @Bean(initMethod = "init", destroyMethod = "close")
    public DruidDataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        // 基本属性 url、user、password
        dataSource.setDriverClassName(environment.getProperty("jdbc.driver"));
        dataSource.setUrl(environment.getProperty("jdbc.url"));
        dataSource.setUsername(environment.getProperty("jdbc.username"));
        dataSource.setPassword(environment.getProperty("jdbc.password"));
        return dataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(ApplicationContext applicationContext) throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource());
        sessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:mapper/*.xml"));
        sessionFactoryBean.setTypeAliasesPackage("com.logic.model");
        return sessionFactoryBean.getObject();
    }
}
````

在此文件中，对Mybatis进行了相关配置

- `@MapperScan`注解扫描dao所在到的包。
- `@PropertySource`用于加载资源文件，代码中的`Environment`结合使用。
- 方法`public DruidDataSource dataSource()`数据库连接池进行配置。
- 方法`public SqlSessionFactory sqlSessionFactory(ApplicationContext applicationContext)`配置`SqlSessionFactory`，并配置`mapper`文件所在的位置，

Maven的配置文件以及其他的源代码文件在附件中，不用细说。