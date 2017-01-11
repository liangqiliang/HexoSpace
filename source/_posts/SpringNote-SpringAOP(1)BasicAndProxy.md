---
title: Spring一知半解-AOP(1)基础知识与动态代理
date: 2017-01-09 15:36:53
categories: [Spring,Java]
---

> AOP，面向切面的编程，一般适用于具有横切逻辑的应用场合，如性能检测、访问控制、事物管理以及日志记录，此文讲述一般基础知识与动态代理。

<!-- more -->

## AOP概述

### 什么是AOP

- 在面向对象的编程语言中，我们通过引用继承来减少或者消除多个类中的重复代码；
- 这种方式并不能处理所有的业务需求；
- 在一些横切业务性代码，如性能监视和事务管理，无法通过继承来消除代码冗余，就需要通过一种新的方式来抽取这些可以独立出来的横切业务代码，并融合到原来的而业务逻辑完成和原来一样的业务操作。
- 以下是具有横切逻辑业务的代码：

``` bash 
public void removeUser(int userId) {
	mornitor.start();
	transManager.beginTransaction();
	userDao.removeUser(userId);
	transManager.commit();
	mornitor.end();
}

public void addUser(User user) {
	mornitor.start();
	transManager.beginTransaction();
	userDao.addUser(user);
	transManager.commit();
	mornitor.end();
}
```

- 上述代码中的性能监视代码和事务代码，在业务代码前后启动和关闭，业务代码被一些重复性的非业务代码所包围，难以直观地体现这段代码的核心业务。

### AOP术语

- **连接点（JoinPoint）**
	- 程序执行的某个特定位置，Spring仅支持方法的连接，即只可以在方法调用前、调用后、抛出异常时这些执行点织入增强

- **切点（PointCut）**
	- 一个类可以拥有多种方法，这些方法都可以是连接点，而AOP通过“切点”来选择特定的连接点，“切点”只定位到某个方法上。一个切点可以匹配多个连接点

- **增强（Advice）**
	- “增强”是织入到目标类连接点上的一段程序代码；“增强”既包含了用于添加到目标连接点上的一段执行逻辑，又包含了用于定位连接点的方位信息

- **目标对象（Target）**
	- “增强”代码的织入目标类

- **引介（Introduction）**
	- 一种特殊的增强；如果一个类没有实现某个接口，可以动态地为该类添加接口的实现逻辑，让业务成为这个接口的实现类

- **织入（Weaving）**
	- “织入”就是“增强”连接到目标连接点上的过程；有三种织入方式：`编译器织入`、`类装载期织入`、`动态代理织入`；Spring采用动态代理织入，而AspectJ采用编译器织入和类装载器织入

- **代理（Proxy）**
	- 一个类织入增强后所产生的结果类，融合了原类和增加逻辑；既可以是与原类具有相同接口的类，也可以是原来的子类；可以采用与原类调用的相同方式调用代理类

- **切面（Aspect）**
	- 由“切点”和增强（引介）组成，既包括横切逻辑的定义，也包括了连接点的定义；`Spring AOP`就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中

## 动态代理

- `Spring AOP`使用了两种代理机制：一种是基于JDK的动态代理；另一种是基于CGLib的动态代理。JDK只提供接口的代理，而不支持类的代理，所以需要使用两种代理。

### JDK动态代理

- 动态代理主要涉及到`java.lang.reflect`的两个类：`Proxy`和`InvocationHandler`；
- `InvocationHandler`是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编织在一起
- UserServiceImpl：移除性能监视横切代码，"//"为移除代码：

``` bash
public class UserServiceImpl implements UserService {
	public void removeUser(int userId) {
		//PerformanceMornitor.begin("com.lql.service.UserService.removeUser");
		System.out.println("Remove the user:" + userId);//模拟删去一个用户
		try {
			Thread.currentThread.sleep(20);
		} catch (Exception e) {
			throw new RuntimeExcption(e);
		}
		//performanceMornitor.end();
	}
}
```

- 性能监视代码PerformanceMornitor：

``` bash
public class PerformanceMornitor {
	public static void begin(String str){
		...begin code
	}
	public static void end(){
		...end code
	}
}
```

- 移除的性能监视代码需要有相应的安置地：

``` bash
import java.lang.reflect.invocationHandler;
import java.lang.reflect.Method;
public class PerformanceHandler implements InvocationHandler {
	private Object target;
	public PerformanceHandler(Object target) {
		this.target = target;
	}
	public Object invoke(Object proxy, Method method, Object[] args) throw Throwable {
		PerformanceMornitor.begin(target.getClass().getName() + "." + method.getName());
		Object obj = method.invoke(target, args);//通过反射方法调用业务类的目标方法
		PerformanceMornitor.end();
		return obj;
	}
}
```

- 这个接口定义了一个`invoke(Object proxy, Method method, Object[] args)`，proxy是最终生成的代理实例，很少用到；method是被代理目标实例的某个具体方法，通过它可以发起目标实例方法的反射调用；args是传给被代理实例的某方法的入参数组，反射调用时使用。
- 创建代理实例：

``` bash
public class Test {
	public void main(String[] args) {
		//定义被代理的目标业务类
		UserService target = new UserServiceImpl();
		//将目标业务类和横切代码编织到一起
		PerfomanceHandler handler = new PerformanceHandler(target);
		//根据handler创建代理实例
		UserService proxy = (UserService) Proxy.newProxyInstance(
			target.getClass().getClassLoader(),
			target.getClass().getInterfaces(),
			handler);
		//调用代理实例
		proxy.removeUser("111");
	}
}
```

### CGLib动态代理

- JDK动态代理只能为接口创建代理实例；并且一个简单的业务都需要创建多个接口，不能直接通过实现类来构建业务；
- `CGLib`采用非常底层的**字节码结束**，可以为一个类创建子类，并在子类中采用**方法拦截**的技术拦截所有父类方法的调用，并乘机织入增强；
- 定义一个代理类，并实现net.sf.cglib.proxy.MethodInterceptor接口：

``` bash
import java.lang.reflect.Method;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.Method;

publc class CglibProxy implements MethodInterceptor {
	private Enhancer enhancer = new Enhancer();
	public Object getProxy(class clazz) {
		//设置需要创建子类的类
		enhancer.setSuperclass(clazz);
		enhancer.setCallback(this);
		//通过字节码技术动态创建子类实例
		return enhancer.create();
	}
	public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {	//拦截父类所有方法的调用
		PerformanceMornitor.begin(target.getClass().getName() + "." + method.getName());
		//通过代理类调用父类的方法
		Object result = proxy.invokeSuper(obj, args);
		PerformanceMornitor.end();
		return result;
	}
}
```

``` bash
import java.lang.reflect.Proxy;
public class Test {
	public void main(String[] args) {
		CglibProxy proxy = new CglibProxy();
		//通过动态生成子类的方法创建代理类
		UserServiceImpl userService = (UserServiceImpl)proxy.getProxy(UserServiceImpl.class);
		proxy.removeUser("111");
	}
}
```

### 代理的不足
- 1.目标类的所有方法都添加了横切逻辑代码；有时我们可能只希望对业务中的某些特定方法添加横切代码；
- 2.通过硬编码的方法指定了织入横切逻辑的织入点；
- 3.为不同类创建代理时，需要分别编写相应的程序代码，不通用。