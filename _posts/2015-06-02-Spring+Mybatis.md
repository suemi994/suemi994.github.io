---
layout: post
title: 从Spring配置说开来
category: WEB开发
tags: java
date: 2015-06-02
---
{% include JB/setup %}


* 目录
{:toc}

### 前言

最近开始尝试使用久负盛名的SM框架——Spring+Mybatis进行Web开发，从此走上了Spring配置的不归路。说句实在话，Spring是我特么见过所有WEB工程里面最难配的，因为必须要对它那一套非侵入的设计思想有所了解才能在出问题时迅速判断是不是配置错误，进而定位配置错误所在。首先，做个简单的总结，Spring的配置现在大致有三种：

- XML配置，时间最长，最主流，也最恶心的配置方法
- 使用配置类+注解，推出有一段时间了，用的人不多，与其他框架集成的时候大部分还是绕不过XML。个人觉得没啥卵用。
- Spring4新推出的Boot，对新手是友好了，但是不利于了解这个框架，而且遇到问题能查的资料少

综上，我最后还是选择了XML作为开发的标配，下面以 Spring+Mybatis配置作为示例

### Maven依赖文件——pom.xml

没啥好讲的，java程序员都知道

~~~XML
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>suemi</groupId>
	<artifactId>sbeat</artifactId>
	<packaging>war</packaging>
	<version>0.0.1-SNAPSHOT</version>
	<name>SBEAT</name>
	<url>http://maven.apache.org</url>

	<properties>
		<mysql.connector.version>5.1.21</mysql.connector.version>
		<spring.version>4.1.6.RELEASE</spring.version>
		<junit.version>4.12</junit.version>
		<velocity.version>1.6.1</velocity.version>
		<velocity.tools.version>2.0</velocity.tools.version>
		<mock.version>2.0.8</mock.version>
		<mybatis.version>3.3.0</mybatis.version>
		<mybatis-spring.version>1.2.2</mybatis-spring.version>
	</properties>

	<dependencies>

		<!-- velocity -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context-support</artifactId>
			<version>${spring.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.velocity</groupId>
			<artifactId>velocity</artifactId>
			<version>${velocity.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.velocity</groupId>
			<artifactId>velocity-tools</artifactId>
			<version>${velocity.tools.version}</version>
		</dependency>

		<!-- jsp support -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>3.1.0</version>
			<scope>provided</scope>
		</dependency>

		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>jstl</artifactId>
			<version>1.2</version> <!-- just used the latest version, make sure you use the one you need -->
			<scope>provided</scope>
		</dependency>

		<!-- test support -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>${junit.version}</version>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-test</artifactId>
			<version>${spring.version}</version>
		</dependency>



		<!-- mybatis support -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis</artifactId>
			<version>${mybatis.version}</version>
		</dependency>

		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis-spring</artifactId>
			<version>${mybatis-spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-dbcp2</artifactId>
			<version>2.0</version>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>${mysql.connector.version}</version>
		</dependency>

		<!-- Spring support -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-orm</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-tx</artifactId>
			<version>${spring.version}</version>
		</dependency>



	</dependencies>
	<build>
		<finalName>sbeat</finalName>
	</build>
</project>

~~~

### web.xml

wweb.xml规定了有哪些显式的servlet，规定了最顶层的Context环境，是整个的纲领。它同时规定了对于哪些URL的请求归哪些servlet处理，哪些URL使用哪种编码方式。需要注意的是，对于没有人为指定配置文件的servlet，会自动在WEB-INF文件夹下找servletName-servlet.xml的文件作为其配置。

~~~XML


<web-app id="WebApp_ID" version="2.4"
 xmlns="http://java.sun.com/xml/ns/j2ee"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee 
 http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
 
  <display-name>sbeat</display-name>
  
  <filter>
		<filter-name>characterEncodingFilter</filter-name>
		<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
		<init-param>
			<param-name>encoding</param-name>
			<param-value>UTF-8</param-value>
		</init-param>
		<init-param>
			<param-name>forceEncoding</param-name>
			<param-value>true</param-value>
		</init-param>
	</filter>

	<filter-mapping>
		<filter-name>characterEncodingFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
	
	
	<servlet>
		<servlet-name>sbeat</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>sbeat</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
	
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath*:applicationContext.xml</param-value>
		<!--指定Context的配置文件-->
	</context-param>
  
  	<listener>
  		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  	</listener>
  
</web-app>

~~~

### URL分发
这里主要就是控制器的扫描，以及配置模板引擎，以velocity模板为例

~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd
      http://www.springframework.org/schema/mvc
      http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">

    <!--这一句处理对静态资源的访问-->
	<mvc:default-servlet-handler />
	<!--这一句指定了扫描注解生成Controller的根包名-->
	<context:component-scan base-package="sbeat" />
	<!--这一句相当于开启注解扫描功能-->
	<mvc:annotation-driven />

	
	<!--规定模板文件的类型和位置-->
	<bean id="velocityConfigurer"
		class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
		<property name="resourceLoaderPath" value="templates/" />
		<property name="velocityProperties">
			<props>
				<prop key="input.encoding">utf-8</prop>
				<prop key="output.encoding">utf-8</prop>
			</props>
		</property>
	</bean>
	<!--配置附加工具，以及将后缀为vm的文件交给下面的Resolver处理-->
	<bean id="velocityViewResolver"
		class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
		<property name="suffix" value=".vm" />
		<property name="contentType" value="text/html;charset=utf-8" />
		<property name="toolboxConfigLocation" value="/WEB-INF/toolbox.xml"/>
	</bean>


</beans>
~~~

toolbox.xml示例如下，您可以选择自己喜欢的模板引擎。

~~~XML
<?xml version="1.0"?>
<toolbox>
    <tool>
        <key>date</key>
        <scope>request</scope>
        <class>
            org.apache.velocity.tools.generic.DateTool
        </class>
        <parameter name="format" value="yyyy-MM-dd HH:mm:ss"/>
    </tool>
    <tool>
        <key>link</key>
        <scope>request</scope>
        <class>org.apache.velocity.tools.view.tools.LinkTool</class>
    </tool>
    <tool>
        <key>stringUtils</key>
        <scope>request</scope>
        <class>org.apache.velocity.util.StringUtils</class>
    </tool>
    <tool>
        <key>math</key>
        <scope>application</scope>
        <class>org.apache.velocity.tools.generic.MathTool</class>
    </tool>
    <tool>
        <key>esc</key>
        <scope>request</scope>
        <class>org.apache.velocity.tools.generic.EscapeTool</class>
    </tool>
    <tool>
        <key>params</key>
        <scope>request</scope>
        <class>org.apache.velocity.tools.view.tools.ParameterParser</class>
    </tool>
</toolbox>
~~~

### applicationContext.xml——最重要的配置文件
applicationContext.xml主要是对数据库相关操作的一些配置，也是Spring与ORM类框架集成的节点。

~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p" 
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	http://www.springframework.org/schema/context 
	http://www.springframework.org/schema/context/spring-context-3.0.xsd 
	http://www.springframework.org/schema/aop 
	http://www.springframework.org/schema/aop/spring-aop-3.0.xsd 
	http://www.springframework.org/schema/tx 
    http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
    http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd"
>
	<tx:annotation-driven transaction-manager="transactionManager"/>
	<!--开启扫面注解生成Bean-->
	<context:annotation-config></context:annotation-config>
	<!--设定扫描的根package-->
	<context:component-scan base-package="sbeat" />
	
	
	<!--这一段是最重要的-->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<!--规定了扫描mapper文件的路径-->
		<property name="mapperLocations" value="classpath*:sbeat/*/mapper/*.xml" />
		<!--指定了Mybatis配置文件的路径-->
		<property name="configLocation" value="classpath:mybatis-config.xml"/>
	</bean>


	<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
		<constructor-arg index="0" ref="sqlSessionFactory" />
	</bean>

	<!--这一段让我们的MapperInterface能够最终在Spring中生成Bean最后被AutoWired-->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="sbeat.*.mapper" />
	</bean>
	

	
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource" />
	</bean>
	
	<!--数据库设置-->
	<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource"
		destroy-method="close">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://localhost:3306/sbeat" />
		<property name="username" value="suemi" />
		<property name="password" value="123456" />
	</bean>
</beans>
~~~

### Mybatis配置

~~~XML
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" 
"http://mybatis.org/dtd/mybatis-3-config.dtd"> 
<configuration>
	<typeAliases>
		<!--POJO 类的所在package，默认类名为其Alias-->
		<package name="sbeat.user.model"/>
	</typeAliases>
	<mappers>
		<!--映射文件所在的地方-->
		<mapper resource="UserMapper.xml" />
	</mappers>
</configuration>	
~~~


### Mybatis映射文件
该文件是将POJO类的

### 使用Spring-test测试

下面给出一个非常简单的测试例子，应该很容易看懂，就不做解释了

~~~java
package seed.test;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import demo.mapper.UserMapper;
import demo.model.User;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:applicationContext.xml"})
public class TsetUserDAO {
	@Autowired
	private UserMapper userMapper;
	
	@Test
	public void testInsert() {
		User user=new User();
		user.setAccount("suemi994");
		user.setPasswd("123456");
		userMapper.addUser(user);
	}
}

~~~

### 小结
Spring常用的注解主要有以下几种：
- @Component
- @Service--业务层
- @Respority--持久化层
- @Controller--视图层

尽管这几种注解对应的意义不一样，但它们都有一个共同的作用，那就是让Spring找到并且最终生成对应的Bean。举个例子，我们在Service中使用接口UserDAO，但该接口的实例我们并没有指定使用哪一个，虽然我们写了对应的Impl Class。 通过注解生成Bean，Spring就能够在Service调用该接口的时候从容器里得到正确的实体对象。

最后还想问一个问题，有什么好方法在使用Mybatis的时候也能根据POJO类的定义改变自动更新数据库结构呢？