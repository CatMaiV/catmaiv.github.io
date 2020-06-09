--- 
layout: post
title: 从头搭建一个SpringBoot+JPA框架
subtitle: 从头搭建一个SpringBoot+JPA框架
date: 2020-06-09
author: catmai
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - SpringBoot
    - JPA
    - Thymeleaf
---





​		最近项目需要一个SpringBoot+JPA的框架做一个简单的系统用于管理数据迁移，选择SpringBoot的原因是配置起来比较简单。JPA的优势在于可以通过很小的改动直接连接Oracle和Mysql移植性比较好。而且因为是比较简单的系统，也不采用前后端分离。



技术总结：

> SpringBoot
>
> JPA
>
> Oracle/Mysql
>
> Thymeleaf
>
> Quartz



​		话不多说，直接上代码。

#### 依赖

​		SpringBoot可以从官网选择需要的组件快速搭建，也可以生成一个maven项目然后配置pom.xml内容引入需要的内容。

``` xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>

		<!--Mysql驱动-->
<!--		<dependency>-->
<!--			<groupId>mysql</groupId>-->
<!--			<artifactId>mysql-connector-java</artifactId>-->
<!--			<scope>runtime</scope>-->
<!--		</dependency>-->
		<!--Oracle驱动-->
		<dependency>
			<groupId>com.oracle</groupId>
			<artifactId>ojdbc7</artifactId>
			<version>12.1.0.2</version>
		</dependency>
		<!-- druid数据源驱动 -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid-spring-boot-starter</artifactId>
			<version>${druid.version}</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>${fastjson.version}</version>
		</dependency>
		<!--监控sql日志-->
		<dependency>
			<groupId>org.bgee.log4jdbc-log4j2</groupId>
			<artifactId>log4jdbc-log4j2-jdbc4.1</artifactId>
			<version>${log4jdbc.version}</version>
		</dependency>
		<!--定时任务-->
		<dependency>
			<groupId>org.quartz-scheduler</groupId>
			<artifactId>quartz</artifactId>
			<exclusions>
				<exclusion>
					<groupId>com.mchange</groupId>
					<artifactId>c3p0</artifactId>
				</exclusion>
			</exclusions>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
```

补一下版本控制

``` xml
<properties>
	<java.version>1.8</java.version>
	<druid.version>1.1.10</druid.version>
	<log4jdbc.version>1.16</log4jdbc.version>
	<fastjson.version>1.2.54</fastjson.version>
</properties>
```

SpringBoot 版本是2.1.0，刚开始用的是最新的2.3.0，发现和druid版本可能有不适配的情况，也不想找了于是采用SpringBoot2.1.0版本。

#### 配置

​		以上依赖载入完毕之后，在application配置下自己的内容。

##### application.yml

``` yaml
server:
  port: 8099
  servlet:
    # 应用的访问路径
    context-path: /
    tomcat:
      # tomcat的URI编码
      uri-encoding: UTF-8
      # tomcat最大线程数，默认为200
      max-threads: 800
      # Tomcat启动初始化的线程数，默认值25
      min-spare-threads: 30

spring:
  thymeleaf:
    mode: HTML
    encoding: utf-8
    # 禁用缓存
    cache: false
  profiles:
    active: dev
  jackson:
    time-zone: GMT+8

  #配置 Jpa
  jpa:
    properties:
      hibernate:
        #dialect: org.hibernate.dialect.MySQL5InnoDBDialect
    open-in-view: true
```

##### application-dev.yml

``` yaml
#配置数据源
spring:
  datasource:
    druid:
      type: com.alibaba.druid.pool.DruidDataSource
      #driverClassName: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
      driverClassName: oracle.jdbc.driver.OracleDriver
      #url: jdbc:log4jdbc:mysql://10.80.18.223:3306/czapp?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useSSL=false
      url: jdbc:oracle:thin:@***:***:***
      username: ***
      password: ***

      # 初始化配置
      initial-size: 3
      # 最小连接数
      min-idle: 3
      # 最大连接数
      max-active: 15
      # 获取连接超时时间
      max-wait: 5000
      # 连接有效性检测时间
      time-between-eviction-runs-millis: 90000
      # 最大空闲时间
      min-evictable-idle-time-millis: 1800000
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false

      validation-query: select 1 from dual  #mysql连接这边貌似可以只写一个select 1
      # 配置监控统计拦截的filters
      filters: stat
      stat-view-servlet:
        url-pattern: /druid/*
        reset-enable: false

      web-stat-filter:
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*"

  #配置 Jpa
  jpa:
    hibernate:
      #设置成 none，避免程序运行时自动更新数据库结构
      ddl-auto: none
    show-sql: true
```

至此，配置全部完成可以正确的启动项目了。

#### 查询

​		因为采用了JPA，所有查询变得十分简单，不需要写Sql了。

​		直接继承JpaRepository和JpaSpecificationExecutor，下面以user表为例

##### UserRepository

``` java
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {

}
```

##### User类

``` java
@Entity
@Getter
@Setter
@Table(name="user")  //表名和数据库表名对应
public class User implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    @NotNull(groups = Update.class)
    private Long id;

    @Column(name = "usercode",nullable = false)
    @NotBlank
    private String usercode;

    @Column(name = "username",nullable = false)
    @NotBlank
    private String username;

    @Column(name = "password",nullable = false)
    private String password;

}
```

##### Service

随手定义一下Service层

``` java
public interface UserService {

    List<User> getAllUser();

}

```

##### ServiceImpl

``` java
@Service(value = "userServiceImpl")
@Transactional
public class UserServiceImpl implements UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public List<User> getAllUser() {
        return userRepository.findAll(); //全表查询
    }
}
```

##### Controller

```java
@Controller
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userServiceImpl;

    @ResponseBody
    @RequestMapping("/list")
    public List<User> List(){
        return userServiceImpl.getAllUser();
    }

}
```

随后启动项目，路由直接敲进list接口，就可以在页面上获得User表下所有内容。



至此整个框架算是搭好了，因为还有需要连接mysql的需求所以保留了一些mysql的东西。后续会进一步研究一下配置双数据源的写法。

















