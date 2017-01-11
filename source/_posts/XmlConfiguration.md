---
title: Spring一知半解-Xml配置
date: 2016-12-30 14:36:53
categories: [Spring,Java]
tags: [Spring,bean,Schema,xml]
---

> Spring配置复杂，寻求一种能够归纳大部分配置的格式文件

<!-- more -->

## Schema格式

### 概述

- 可扩展标记语言架构是以可扩展标记语言（标准通用标记语言的子集）为基础的，它用于可替代文档类型定义（外语缩写：DTD）；一份XML schema文件描述了可扩展标记语言文档的结构。
- 一份XML Schema定义了：
	- 可以出现在文档里的元素；
    - 可以出现在文档里的属性；
    - 哪些元素是子元素；
    - 子元素的顺序；
    - 子元素的数量；
    - 一个元素应是否能包含文本，或应该是空的；
    - 元素和属性的数据类型；
    - 元素和属性的默认值和固定值。

## XML配置文件

### Spring配置详解

- ![Spring配置详解1](http://img.blog.csdn.net/20130918150129015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjA0OTQ2Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- ![Spring配置详解2](http://img.blog.csdn.net/20130918150250171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjA0OTQ2Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- ![Spring配置详解3](http://img.blog.csdn.net/20130918150314578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjA0OTQ2Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- ![Spring配置详解4](http://img.blog.csdn.net/20130918150327968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjA0OTQ2Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- ![Spring配置详解5](http://img.blog.csdn.net/20130918150343562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjA0OTQ2Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Spring配置代码

``` bash
	<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:task="http://www.springframework.org/schema/task"
       xsi:schemaLocation="http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.1.xsd
		http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.2.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.1.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd">
		<!--开启注解处理器-->
		<context:annotion-config/>
		<!--开启组件自动扫描，扫描路径由base-package指定-->
		<context:compoment-scan base-package="test.spring">
		<!--开启基于@Aspectj切面的注解处理器-->
		<aop:aspectj-autoproxy/>
		<!--使用class属性指定的类的默认构造方法创建一个单实例Bean，名称由ID属性指定。-->
		<bean id="Bean实例名称" class="Bean类的全限定路径">
        <!--Scope属性为prototype时表示每次将生成新的实例，即原型模式。-->
		<bean id="Bean实例名称" class="Bean类的全限定路径" scope="prototype" >
        <!--Init-method属性用于指定对象实例化后要调用的初始化方法。-->
        <!--Destory-method属性用于指定对象在销毁时要调用的方法。-->
		<bean id="Bean实例名称" class="Bean类的全限定路径" init-method="初始化时调用的方法"  desctory-method="对象销毁时调用的方法" >
		<bean id="Bean实例名称" class="Bean类的全限定路径">
        	<!--Property标签用于对Bean实例中的属性进行赋值，对于基本数据类型的值可以直接用value属性指定，而其他bean引用则可以使用ref。-->
    		<property name="Bean类中的属性名称" ref=“要引用的Bean名称"/>
    		<property name="Bean类中的属性名称" value=“直接指定属性值"/>
        	<!--创建一个内部匿名Bean实例赋值给指定的属性，该匿名实例无法被外界访问。-->
    		<property name="Bean类中的属性名称" >
        		<bean class="Bean类全限定路径"/>
        	</propertype>
			<!--Set标签用于创建一个Set类型的实例赋值给指定的Set类型属性，Set实例中的元素通过Value或者ref子标签指定，对于基本数据类型可以使用value标签，如果是其他的Bean类实例作为Set元素则需要使用ref标签指定。-->
			<property name="Bean类中的Set属性名称">
				<set>
					<value>set中的元素</value>
					<ref bean="要引用的Bean名称"/>
				</set>
			</property>
		<!--Map标签用于创建一个map类型的实例赋值给指定的额Map类型属性，Map实例中的元素通过entry子标签来指定，Map元素的键由entry标签的key属性指定，值由value或者ref来指定。-->
			<property name="Bean类中的Map类型属性名称">
				<map>
					<entry key="map元素的key">
						<value>map元素的value</value>
					</entry>
					<entry key="map元素的key">
						<ref bean="要引用的bean名称"/>
				</map>
			</property>
			<property name="Bean类中的Proiperties类型的属性名称">
				<props>
					<prop key="properties元素的key">Properties元素的value</prop>
        		</props>
        	</property>
        	<!-- 创建一个properties类型的实例赋值给指定的Properties类型属性，
        	Properties实例中属性项由prop标签生成，属性项元素的键由key属性指定，属性项元素的值可直接放置在prop标签体中。 -->
			<property name="">
				<!-- Null标签用于给需要赋null值的属性进行赋null值。 -->
				<null/>
        	</property>
        </bean>
		<!-- 通过传入相应的构造参数进行Bean实例化，constructor-arg标签用于制定一个构造参数，其index属性表明当前是第几个构造参数。
        Type属性声明构造参数的类型，构造参数的值如果是基本类型可由vlue直接指定，如果是对象的引用，则由ref指定。 -->
        <bean id="Bean实例名称" class="Bean类全名">
			<construct-arg index="从0开始的序号" type="构造参数的类型" value="构造参数的值"/>
			<construct-arg index="从0开始的序号" type="构造参数的类型" ref="要引用的bean名称"/>
        </bean>
        <aop:config>
			<aop:aspect id="切面ID" ref="要引用的切面实例名称">
				<aop:pointcutid="切入点名称" expression="切入点正则表达式"/>
				<aop:before pointcut-ref="切入点名称" method="切面类中用作前置通知的方法名"/>
				<aop:after-returning pointcut-ref="切入点名称" method="切面类中用作后置通知的方法名"/>
				<aop:after-throwing pointcut-ref="切入点名称" method="切面类中用作异常通知的方法名"/>
				<aop:after pointcut-ref="切入点名称" method="切面类中用作最终通知的方法名"/>
				<aop:around pointcut-ref="切入点名称" mthod="切面类中用作环绕通知的方法名"/>
			</aop:aspect>
        </aop:config>
        <!-- 配置事务管理器 -->
        <bean id="事务管理器实例名称" class="事务管理器的全限定名称">
			<property name="数据源属性名称" ref="要引用的数据源实例名称"/>
		</bean>
		<!-- 配置一个事务通知 -->
		<tx:advice id="事务通知名称" transaction-manager="事务管理器实例名称">
			<tx:attributes>
				<tx:method name="get*" read-only="true" propagation="NOT_SUPPORTED"/>
				<tx:method name="*"/>
			</tx:attributes>
        </tx:advice>
        <!-- 使用AOP技术实现事务管理 -->
        <aop:config>
			<aop:pointcut id="事务切入点名称" expression="事务切入点正则表达式"/>
			<aop:advisor advice-ref="事务通知名称" pointcut-ref="事务切入点名称"/>
		</aop:config>
	</beans>
```