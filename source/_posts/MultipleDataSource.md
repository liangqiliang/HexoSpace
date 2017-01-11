---
title: Spring一知半解-基于注解多数据源切换及配置问题
date: 2017-01-04 15:39:53
categories: [Spring,Java]
tags: [Spring,数据库,数据源,xml,AOP]
---

> 本文将介绍在多个数据源的前提下实现数据源的动态切换

<!-- more -->


## 多数据源

### 数据源

- Java操作数据库时的一种重要的技术，流行的持久化框架都离不开数据源的应用；
- 提供了一种简单获取数据库连接的方式，并在内部以池的机制来复用数据库连接，大大减少创建数据库连接的次数，提高了系统性能；
- 用途较广的开源数据源或数据库连接池有：DBCP、C3P0、Proxool、Druid。

### 多数据源

- 一般而言，一个数据库实例代表一个数据源；
- 多数据源一般用于下面几种场景：
	- 1.项目设计上需要多个数据源的数据；
	- 2.大型应用中对数据进行切分，并且采用多个数据库实例进行管理，这样可以有效提高系统的水平伸缩性；
	- 3.网站达到一定规模后，为了改善数据库负载压力，通过配置主从关系数据库进行数据库的读写分离。
- 利用Spring对数据源进行配置，利用AOP面向切面编程实现数据库的动态切换，利用注解使得程序在运行时根据当时的请求及系统状态来动态的决定将数据存储在哪个数据库实例中，以及从哪个数据库提取数据。


## 多数据源实现及配置

### 数据源配置

- 在`spring-config`中定义及配置一个数据源，其他数据源配置类似，只是属性不相同:

``` bash
<!-- 读取jdbc属性文件 -->
<context:property-placeholder location="classpath:jdbc.properties"/>
<!--数据源1-->
    <bean id="DataSource1" class="com.alibaba.druid.pool.DruidDataSource" init-method="init"
          destroy-method="close">
        <!-- 基本属性 url、user、password -->
        <property name="url" value="${mysql.url}"/>
        <property name="username" value="${mysql.username}"/>
        <property name="password" value="${mysql.password}"/>
        <property name="driverClassName" value="${mysql.driverClassName}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="${druid.initialSize}"/>
        <property name="minIdle" value="${druid.minIdle}"/>
        <property name="maxActive" value="${druid.maxActive}"/>

        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="${druid.maxWait}"/>
        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="${druid.timeBetweenEvictionRunsMillis}"/>

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="${druid.minEvictableIdleTimeMillis}"/>

        <property name="validationQuery" value="${druid.validationQuery}"/>
        <property name="testWhileIdle" value="${druid.testWhileIdle}"/>
        <property name="testOnBorrow" value="${druid.testOnBorrow}"/>
        <property name="testOnReturn" value="${druid.testOnReturn}"/>

        <!-- 是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭。-->
        <property name="poolPreparedStatements" value="false"/>
        <property name="maxPoolPreparedStatementPerConnectionSize"
                  value="${druid.maxPoolPreparedStatementPerConnectionSize}"/>

        <!-- 配置监控统计拦截的filters -->
        <property name="filters" value="${druid.filters}"/>
    </bean>
<!--数据源1 END-->
```

- 定义多数据源`MultipleDataSource`,实现`AbstractRoutingDataSource`抽象数据源中方法:
- `determineCurrentLookupKey()`方法，这是AbstractRoutingDataSource类中的一个抽象方法，而它的返回值是你所要用的数据源dataSource的key值，有了这个key值，`resolvedDataSource`（这是个map,由配置文件中设置好后存入的）就从中取出对应的DataSource，如果找不到，就用配置默认的数据源。

``` bash
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
public class MultipleDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceTypeManager.getDataSourceType();
    }
}
```

- 在`spring-config`中配置`MultipleDataSource`

``` bash
<!-- 配置可以存储多个数据源的Bean -->
<bean id="multipleDataSource" class="com.eking.emon.commons.peresist.MultipleDataSource">
    <property name="defaultTargetDataSource" ref="emonDataSource" />
    <property name="targetDataSources">
        <map key-type="java.lang.String">
            <entry key="source1" value-ref="DataSource1"/>
            <!--<entry key="hr" value-ref="hrDataSource"/>-->
            <entry key="source2" value-ref="DataSource2"/>
            <!-- 这里还可以加多个dataSource -->
        </map>
    </property>
</bean>
 ```

### 获取及切换数据源

- 定义`DataSourceTypeManager`来获取当前线程的数据源及动态切换数据源:

``` bash
public class DataSourceTypeManager {

    private static final ThreadLocal<String> dataSourceTypes = new ThreadLocal<String>();

    public static String getDataSourceType() {
        return dataSourceTypes.get();
    }

    public static void setDataSourceType(String dataSourceType) {
        dataSourceTypes.set(dataSourceType);
    }

    public static void clearDataSourceType() {
        dataSourceTypes.remove();
    }
}
```

### SqlSessionFactory

- 配置`SqlSessionFactory`

``` bash
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="multipleDataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    <property name="configLocation" value="classpath:/mybatis-config.xml"/>
    <property name="typeAliasesPackage" value="com.eking.emon.farm.model"/>
    <property name="plugins">
        <array>
            <ref bean="pageInterceptor"/>
            <ref bean="sqlServerPageInterceptor"/>
        </array>
    </property>
</bean>
```

## Annotation及配置AOP

### 定义注解
- 定义一个数据源注解，指示当前需要访问哪个数据库:

``` bash
import java.lang.annotation.*;
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE,ElementType.METHOD})
@Documented
public @interface DataSource {
    String name();
}
```

### 定义切入点及切面

- 在`spring-config`中定义一个切入点

``` bash
<aop:pointcut id="serviceWithAnnotation"
                expression="@annotation(com.eking.emon.farm.web.annotation.DataSource)" />
```

- 接着定义一个切面

``` bash
<aop:advisor advice-ref="dataSourceInterceptor" pointcut-ref="serviceWithAnnotation" order="1"/>
```

### 定义一个数据库拦截器

- 定义拦截器，使得发现注解`DataSource`时自动拦截，并调用切换数据库的方法

``` bash
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
public class DataSourceInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        DataSource dataSource = this.getDataSource(invocation);
        String dbnameString = dataSource.name();
        Object result;
        try {
            DataSourceTypeManager.setDataSourceType(dbnameString);
            result = invocation.proceed();
        } finally {
        	//最终指向默认数据源
            DataSourceTypeManager.setDataSourceType("source1");
        }
        return result;
    }
	//获取当前注解指示的数据源
    private DataSource getDataSource(MethodInvocation invocation) throws Throwable {
        //TODO
        return invocation.getMethod().getAnnotation(DataSource.class);
    }
}
```

### 配置拦截器

- 在`spring-config`中配置`dataSourceInterceptor`

``` bash
<!-- 配置切换数据源Key的拦截器，用于切换数据源Key -->
    <bean id="dataSourceInterceptor" class="com.eking.emon.commons.peresist.DataSourceInterceptor" />
```

### 使用

- 在需要进行数据库连接的服务具体方法上标注`@DataSource(name= "source1")`，即可实现数据库的切换
``` bash
@DataSource(name= "source1")
public User queryUser(Map<String, Object> paraMap) {
	UserDao userDao = (UserDao) super.crudDao;
	return userDao.queryUser(paraMap);
}
```

## 参考 致谢
- [1.Spring 管理配置多个数据源 ](https://www.oschina.net/question/54100_30592)
- [2.Spring中AOP方式实现多数据源切换](http://www.jianshu.com/p/ddebf4ae57c1)
- [3.Alter dataSource in Spring By AOP And Annotation](Alter dataSource in Spring By AOP And Annotation)