---
date: 2021-01-05T15:23:23+08:00
featured_image: "https://images.unsplash.com/photo-1647163927803-2df53b84969b?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2662&q=80"
tags: [spring]
title: "SSM整合：分层配置和整合"
categories: Java
description: 详细介绍经典SSM框架整合和基础开发环境搭建，MVC分层配置清晰详细
---

## 创建 WEB 项目

---

### Web模块支持

- 在 IDEA 中创建 Maven 项目，添加 web模板，或在一个普通 Maven 项目中添加 Web模块支持
- 右键项目 Add Framework Support
  ![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202203141023428.png)

#### 配置文件准备

- [可选]mybatis-config.xml
	- Mybatis 核心配置文件，最终会整合到 Spring 中配置	 
- spring-dao：Dao 层 整合 Mybatis 层
	- DTO 约束，同 spring-service
    ``` xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/springbeans.
    xsd">
    </beans>
    ```

- spring-service：Service 层，Spring 和配置文件
- spring-mvc：Controller 层，SpringMVC 核心配置文件
	- DTO 约束
    ``` xml
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
           https://www.springframework.org/schema/mvc/spring-mvc.xsd">
          </beans>
    ```
- applicationContext.xml：总核心配置文件，引入各层配置
  ``` xml
    <import resource="classpath:spring-dao.xml"/>
    <import resource="classpath:spring-service.xml"/>
    <import resource="classpath:spring-mvc.xml"/>
  ```

### Maven 静态资源导出（可选）

让 Java 目录下和 Resources 目录下的配置文件都能编译导出
``` xml
  <resources>
      <resource>
          <directory>src/main/java</directory>
          <includes>
              <include>**/*.properties</include>
              <include>**/*.xml</include>
          </includes>
          <filtering>false</filtering>
      </resource>
      <resource>
          <directory>src/main/resources</directory>
          <includes>
              <include>**/*.properties</include>
              <include>**/*.xml</include>
          </includes>
          <filtering>false</filtering>
      </resource>
  </resources>
```

## Sping 整合Mybatis

### Mybatis 环境

#### 相关坐标

``` xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.15</version>
</dependency>
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.1</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
</dependency>
```

#### 数据源配置

数据源配置文件 `datasource.properties`
```properties
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/test?useSSL=true&useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
jdbc.username=root
jdbc.password=jjbbkk123
```
#### （可选）Mybatis 核心配置文件 
```xml
	  <?xml version="1.0" encoding="UTF-8" ?>
	  <!DOCTYPE configuration
	  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
	  "http://mybatis.org/dtd/mybatis-3-config.dtd">
	  <configuration>
	       <!--配置LOG4J-->
	      <settings>
	          <setting name="logImpl" value="log4j"/>
	      </settings>
	  
	      <!--类型别名配置-->
	      <typeAliases>
	      	<package name="com.xxx.domain"/>
	      </typeAliases>
	    
	     <!--properties加载-->
	     <!--数据源配置-->
	      
	      <!-- Mapper.xml 映射文件的配置-->
	  </configuration>
```

### Spring 环境
#### 相关坐标
```xml
	  <!--spring坐标-->
	  <dependency>
	    <groupId>org.springframework</groupId>
	    <artifactId>spring-context</artifactId>
	    <version>5.1.5.RELEASE</version>
	  </dependency>
	  <dependency>
	    <groupId>org.aspectj</groupId>
	    <artifactId>aspectjweaver</artifactId>
	    <version>1.8.13</version>
	  </dependency>
	  <dependency>
	    <groupId>org.springframework</groupId>
	    <artifactId>spring-jdbc</artifactId>
	    <version>5.1.5.RELEASE</version>
	  </dependency>
	  <dependency>
	    <groupId>org.springframework</groupId>
	    <artifactId>spring-tx</artifactId>
	    <version>5.1.5.RELEASE</version>
	  </dependency>
	  <dependency>
	    <groupId>org.springframework</groupId>
	     <artifactId>spring-test</artifactId>
	    <version>5.1.5.RELEASE</version>
	  </dependency>
```
#### Spring 核心配置文件
- 开启注解扫描
  ① 只扫描 service 下的包
  ``` xml
  <context:component-scan base-package="com.zackyj.service" />
  ```
  ② 扫描基包，开启过滤器排除 controller
  ``` xml
  <context:component-scan base-package="com.zackyj.service" use-default-filters="true">
  <context:exclude-filter type="annotation" expression="org.springframework.steretype.Controller"/>
  </context:component-scan> 
  ```
- (注入 bean 或用注解方式)
- 配置事务管理器 和 声明式事务
  ``` xml
  <!-- 配置事务管理器 -->
  <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <!-- 注入数据库连接池 -->
  <property name="dataSource" ref="dataSource" />
  </bean>
  <!--开启事务注解支持-->
  <tx:annotation-driven/>
  ```
#### Spring Junit 测试配置
```java
	  @RunWith(SpringJUnit4ClassRunner.class)
	  @ContextConfiguration(locations = {"classpath:applicationContext.xml"})
	  public class SpringTestor {
	      @Resource
	      private UserService userService;
	  
	      @Test
	      public void testUserService(){
	          userService.createUser();
	      }
	  }
```
### Mybatis 整合
> 将 mybatis 接口代理对象的创建权交给 spring 管理
- 导入整合包依赖

	``` xml
		  <dependency>
		      <groupId>org.mybatis</groupId>
		      <artifactId>mybatis-spring</artifactId>
		      <version>1.3.1</version>
		  </dependency>
	```
- 编写 spring-dao.xml
	关联数据源配置文件 datasourse.properties
  ```xml
    <context:property-placeholder
    location="classpath:datasourse.properties"/>
  ```
	- 在数据库连接池中配置数据源
		- dbcp 半自动化操作 不能自动连接
		``` xml
				  <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
				      <property name="driverClassName" value="${jdbc.driver}"/>
				      <property name="url" value="${jdbc.url}"/>
				      <property name="username" value="${jdbc.username}"/>
				      <property name="password" value="${jdbc.password}"/>
				  </bean>
		```
	- c3p0 自动化操作（自动的加载配置文件 并且设置到对象里面）
		
      ```xml
            <bean id="dataSource"
                  class="com.mchange.v2.c3p0.ComboPooledDataSource">
                <!-- 配置连接池属性 -->
                <property name="driverClass" value="${driver}"/>
                <property name="jdbcUrl" value="${url}"/>
                <property name="user" value="${username}"/>
                <property name="password" value="${password}"/>
                <!-- c3p0连接池的私有属性 -->
                <property name="maxPoolSize" value="30"/>
                <property name="minPoolSize" value="10"/>
                <!-- 关闭连接后不自动commit -->
                <property name="autoCommitOnClose" value="false"/>
                <!-- 获取连接超时时间 -->
                <property name="checkoutTimeout" value="10000"/>
                <!-- 当获取连接失败重试次数 -->
                <property name="acquireRetryAttempts" value="2"/>
            </bean>
      ```
- 配置 SqlSessionFactory 对象交给 spring  IoC 容器
  ```xml
            <bean id="sqlSessionFactory"
                  class="org.mybatis.spring.SqlSessionFactoryBean">
                <!-- 注入数据库连接池 -->
                <property name="dataSource" ref="dataSource"/>
               <!--类型别名也可以在这里配置-->
                <property name="typeAliasesPackage" value="com.lagou.domain"/>
                <!-- 也可以：配置MyBaties全局配置文件:mybatis-config.xml -->
                <property name="configLocation" value="classpath:mybatis-config.xml"/>
            </bean>
  ```
- 配置 Dao 接口扫描
  ``` xml
      <!-- 配置扫描Dao接口包，动态实现Dao接口注入到spring容器中，让Dao接口注入Service-->
      <!--解释 ： https://www.cnblogs.com/jpfss/p/7799806.html-->
      <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
      <!-- 注入sqlSessionFactory -->
      <property name="sqlSessionFactoryBeanName"
      value="sqlSessionFactory"/>
      <!-- 给出需要扫描Dao接口包 -->
      <property name="basePackage" value="com.zackyj.dao"/>
      </bean>
  ```
## 整合SpringMVC

### 相关坐标

```xml
	  <dependency>
	      <groupId>org.springframework</groupId>
	      <artifactId>spring-webmvc</artifactId>
	      <version>5.1.5.RELEASE</version>
	  </dependency>
	  <dependency>
	      <groupId>javax.servlet</groupId>
	      <artifactId>javax.servlet-api</artifactId>
	      <version>3.1.0</version>
	      <scope>provided</scope>
	  </dependency>
	  <dependency>
	      <groupId>javax.servlet.jsp</groupId>
	      <artifactId>jsp-api</artifactId>
	      <version>2.2</version>
	      <scope>provided</scope>
	  </dependency>
	  <dependency>
	      <groupId>jstl</groupId>
	      <artifactId>jstl</artifactId>
	      <version>1.2</version>
	  </dependency>
```
### web.xml 配置

- 配置核心的前端控制器 DispathcerServlet
	``` xml
		  <!--DispatcherServlet-->
		  <servlet>
		     <servlet-name>DispatcherServlet</servlet-name>
		     <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		     <init-param>
		         <param-name>contextConfigLocation</param-name>
		         <!--一定要注意:这里加载的是总的配置文件-->
		         <param-value>classpath:applicationContext.xml</param-value>
		     </init-param>
		     <load-on-startup>1</load-on-startup>
		  </servlet>
		  
		  <servlet-mapping>
		     <servlet-name>DispatcherServlet</servlet-name>
		     <url-pattern>/</url-pattern>
		  </servlet-mapping>
	```
- 配置 Post 请求乱码过滤器
	``` xml
		  <filter>
		     <filter-name>encodingFilter</filter-name>
		     <filter-class>
		        org.springframework.web.filter.CharacterEncodingFilter
		     </filter-class>
		     <init-param>
		         <param-name>encoding</param-name>
		         <param-value>utf-8</param-value>
		     </init-param>
		  </filter>
		  <filter-mapping>
		     <filter-name>encodingFilter</filter-name>
		     <url-pattern>/*</url-pattern>
		  </filter-mapping>
	```
### SpringMVC核心配置文件

```xml
	  <!--组件扫描-->
	  <context:component-scan base-package="com.lagou.controller"/>
	  <!--组件扫描 方式二-->
	  <context:component-scan base-package="com.zackyj" use-default-filters="false">
	      <context:include-filter type="annotation" expression="org.springframework.steretype.Controller"/>
	  </context:component-scan> 
	  <!--mvc注解增强-->
	  <mvc:annotation-driven/>
	  <!--mvc注解增强 解决响应体乱码-->
	     <mvc:annotation-driven>
	      <mvc:message-converters>
	          <bean class="org.springframework.http.converter.StringHttpMessageConverter">
	              <property name="supportedMediaTypes">
	                  <list>
	  <!--                        解决响应体中文乱码-->
	                      <!-- response.setContentType("text/html;charset=utf-8") -->
	                      <value>text/html;charset=utf-8</value>
	                      <value>application/json;charset=utf-8</value>
	                  </list>
	              </property>
	          </bean>
	      </mvc:message-converters>
	  </mvc:annotation-driven>
	  <!--视图解析器-->
	  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
	      <property name="prefix" value="/"/>
	      <property name="suffix" value=".jsp"/>
	  </bean>
	  <!--或返回JSON参数—fastjson-->
	  <mvc:annotation-driven>
	      <mvc:message-converters>
	          <bean class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter">
	          </bean>
	      </mvc:message-converters>
	  </mvc:annotation-driven>
	  <!--实现静态资源映射-->
	  <mvc:default-servlet-handler/>
```
## 整合 SpringMVC

- 无需额外整合，本就是一家
- 做到 spring 和 web 容器整合 (生命周期关联)
	- 让web容器启动的时候自动加载spring配置文件，web容
	- 器销毁的时候spring的ioc容器也销毁
	- 使用监听器来监听 servletContext 容器的创建和销毁
- web.xml 中配置监听器
  
``` xml
  <listener>
    <listener-class>
    	org.springframework.web.context.ContextLoaderListener
    </listener-class>
  </listener>
  <context-param>
  	<param-name>contextConfigLocation</param-name>
  	<param-value>classpath:applicationContext.xml</param-value>
  </context-param>
```