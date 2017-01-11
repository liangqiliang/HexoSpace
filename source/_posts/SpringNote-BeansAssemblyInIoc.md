---
title: Spring一知半解-在IoC容器中装配Bean
date: 2017-01-05 16:36:53
tags: [Spring,bean,依赖,xml]
---

> 对于Spring来说，提供了基于XML、基于注解、基于Java类三种方式在IoC容器中装配Bean

<!-- more -->

## 基于XML的配置

### 命名空间

- 指定命名空间的`Schema`文件地址有两个用途：
	- 1.XML解析器可以获取`Schema`文件并对文档进行格式合法性验证；
	- 2.在开发环境下，IDE通过引用Schema文件可以对文档编辑提供提示诱导功能。
- Spring配置的升级是向前兼容的，最好使用最新的配置说明。

### Bean的命名

- 较为简单，不再细谈，讲几点注意的：
	- id在IoC容器中必须是唯一的，而name可以不唯一；相同name的情况下，后面的Bean会覆盖前面同名的Bean，所以为了防止覆盖产生的隐患，尽量使用id；
	- id和name都未指定时，自动将全限定类名作为Bean的名称；如，`getBean(
	"com.lql.util.Car")`

### 依赖注入

#### 属性注入

- 通过`Setter`方法注入Bean的属性值或依赖对象；
- 要求Bean提供一个默认的构造函数，并为需要注入的属性提供`Setter`方法；

``` bash
<bean id="car" class="com.lql.util.Car">
	<property name="color"><value>![CDATA[red&bule]]</value></property>
	<property name="price"><value>2000</value></property>
</bean>
```

- 在变量命名的时候，需要注意一条规范**“变量的前两个字母要么全部大写，要么全部小写”**，避免因为`Setter`方法不正确而导致配置错误。

- `![CDATA[XXX]]`的作用是让XML解析器将标签中的字符串当做普通的文本对待，以防止某些字符对XML格式的破坏；
- 如果不使用`![CDATA[XXX]]`，则需要另外的转义字符：

特殊符号 | 转义序列 
----|------
> | &lt;  
< | &gt;
& | &amp;
" | &quot;
' | &apos;

#### 构造函数注入

- 用途：如果某一个对象必须提供两个及以上的属性值，通过属性注入方法只能在配置时提供保证，而无法在语法级提供保证；
- 使用构造函数注入的前提是Bean必须提供带参的构造函数

``` bash
public class Car {
	...
	public Car(String color, int price) {
		this.color = color;
		this.price = price;
	}
}
```

``` bash
<bean id="car" class="com.lql.util.Car">
	<construct-arg type="java.lang.String"><value>red</value></property>
	<construct-arg type="int"><value>2000</value></property>
</bean>
```

- 在只有一个构造函数的情况下，`<construct-arg>`声明顺序可以用于确定构造函数入参的顺序；
- 由于Spring使用Java反射机制调用Setter方法完成属性注入，但不会记住构造函数的入参名，只能通过入参类型和索引信息间接确定构造函数配置项和入参的对应关系，如：

``` bash
<bean id="car" class="com.lql.util.Car">
	<construct-arg index="0" type="java.lang.String"><value>red</value></property>
	<construct-arg index="1" type="int"><value>2000</value></property>
</bean>
```

- 当有多个构造函数，这些个构造函数某些参数类型不一致，只需要明确指定不一致参数的类型就可以避免冲突：

- 当如果Bean的构造函数入参类型可辨别，意思是非基础数据类型且类型互不相同，配置里不提供类型和索引，也可以完成注入：

``` bash
public class Desk {
	...
	public Car(String color, Book book, Pen pen) {
		this.color = color;
		this.book = book;
		this.pen = pen;
	}
}
```

``` bash
<bean id="Desk" class="com.lql.util.Desk">
	<construct-arg><value>red</value></property>
	<construct-arg><ref bean="book"/></property>
	<construct-arg><ref bean="pen"/></property>
</bean>
```

- 循环依赖问题：两个Bean都采用构造函数注入，并且参数引用彼此，就会发生类似于线程死锁的循环依赖问题；
**解决方法**： 将构造函数注入方式调整为属性注入方式

#### 工厂方法注入

- 非静态工厂方法：

``` bash
public class CarFactory {
	public Car createOneCar() {
		Car car = new Car();
		car.setColor("blue");
		return car;
	}
}
```

``` bash
<bean id="carFactory" class="com.lql.factory.CarFactory">
<bean id="car1" factory-bean="carFactory" factory-method="createOneCar"/>
```

- 静态工厂方法

``` bash
public class CarFactory {
	public static Car createOneCar() {
		Car car = new Car();
		car.setColor("blue");
		return car;
	}
}
```

``` bash
<bean id="car2" factory-bean="com.lql.factory.CarFactory" factory-method="createOneCar"/>
```

- 因为工厂方法需要额外的类和代码，这些功能和业务是没有关系的，Spring容器已经有一种更好的方式实现了传统工厂模式的所有功能，所以不需要我们再去做类似的工作。

### Bean之间的关系

#### 继承

- 如果多个Bean之间存在有相同的配置信息，可以定义一个父`<bean>`，子`<bean>`将自动继承父`<bean>`的配置信息：

``` bash
<bean id="car1" class="com.lql.util.Car" p:price="2000" p:color="黑色"/>
<bean id="car2" class="com.lql.util.Car" p:price="2000" p:color="白色"/>
```

- 上面两个Bean存在重复信息，可以定义一个父`<bean>`：

``` bash
<bean id="abstractCar" class="com.lql.util.Car" p:price="2000" p:color="黑色" abstract="true"/>
<bean id="car1" p:color="黑色" parent="abstractCar"/>
<bean id="car2" p:color="白色" parent="abstractCar"/>
```

#### 依赖

- 可以使用`<ref>`元素标签来建立对其他Bean的依赖关系;
- 当实例化一个Bean时，Spring保证该Bean所以来的其他Bean已经初始化；

``` bash
<bean id="manager" class="com.lql.util.Manager" depends-on="init"/>
<bean id="init" class="com.lql.util.Init"/>
```

- 通过`depends-on`来指定Bean前置依赖的Bean。

#### 引用

- 虽然可以通过Bean的属性值来指定引用，但是两者没有建立引用关系，在一个Bean中引用另外Bean的id是希望在运行期通过getBean(name)来获取对应的Bean；
- 通过标签`<idref>`来引用另一个Bean：

``` bash
<bean id="pen" class="com.lql.util.Pen"/>
<bean id="desk" class="com.lql.util.Desk">
	<property name="penId">
		<idref bean = "pen">
	</property>
</bean>
```

### 多个配置文件整合

- 启动Spring容器时，可以通过`<import>`将多个配置文件引入到一个文件中，进行配置文件的集成：

``` bash
<import resource="classpath:com/lql/xml/beans1.xml"/>
```

- 当一个配置文件中的Bean引用另一个配置文件中Bean时，并不一定需要`<import>`来引入这个配置文件；只需要在启动容器时，两个配置文件都在列表中，会自动在内存中合并他们。

## 基于注解的配置

### 使用注解定义Bean

- `@Component`
- `@Repository`：用于对DAO实现类进行标注
- `@Service`： 用于对Service实现类进行标注
- `@Controller`：用于对Controller实现类进行标注
- 上面四个注解基本等效，Spring容器自动将`POJO类`转换为容器管理的Bean;
- 在配置文件中加入自动扫描类包中有注解的Bean的配置信息：

``` bash
<context:component-scan base-package="com.lql.util">
```
- `bas-package`属性指定一个需要扫描的基类包，会自动扫描所有类，并从类的注解信息中获取Bean的定义信息；
- 使用`resource-pattern`属性过滤出特定的类：

```bash
<!-- 仅会扫描基包里util里子包中的类 -->
<context:component-scan base-package="com.lql" resource-pattern="util/*.class"
```
- 也可以使用`<context:include-filter>`或者`<context:exclude-filter>`进行过滤，前者表示包含的目标类，后者表示要排除在外的目标类：

``` bash
<context:component-scan base-package="com.lql">
	<context:include-filter type="regex" expression="com\.lql\.util.*"/>
	<context:exclude-filter type="aspectj" expression="com.lql..*Controller+"/>
</context:component-scan>
```
- 更多的关于过滤表达式的细节，以后再深究。

### 自动配置Bean

#### @Autowired
- 通过`@Autowired`注解实现Bean的依赖注入：

``` bash
@Service
public class myService {
	@Autowired
	private UserDao userDao;
}
```
- `@Autowired`默认按类型匹配的方式，在容器查找匹配的Bean，当有且仅有一个匹配的Bean时，Spring将其注入到@Autowired标注的变量中；
- 没有发现匹配的Bean时，抛出`NoSuchBeanDefinitionException`的异常；
- 如果希望不抛出异常，则可以使用`@Autowired(required=false)`进行标注。

#### @Qualifier

- 如果容器中有一个以上可以匹配的Bean时，可以通过`@Qualifier`限定Bean的名称：

``` bash
@Service
public class myService {
	@Autowired(required=false)
	@Qualifier("userDao")
	private UserDao userDao;
}
```

#### 对类方法进行标注

- `@Autowired`也可以对类方法标注入参，通过`@Qualifier`限定Bean的名称：

``` bash
@Service
public class myService {
	private UserDao userDao;

	@Autowired(required=false)
	@Qualifier("userDao")
	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```

#### 对集合类进行标注

- `@Autowired`也可以对类中集合类的变量或方法标注，容器中类型匹配的所有Bean都自动注入进来：

``` bash
@Component
public class MyComponent {
	@Autowired(required=false)
	private List<Plugin> plugins;
}
```

#### @Resource @Inject

- `@Resource @Inject`与`@Autowired`功能类似，不同的是：
- `@Resource`要求提供一个Bean名称的属性：

```bash
@Resource("userDao")
	private UserDao userDao;
```
- 如果没有属性，则自动采用标注出的变量名或方法名作为Bean的名称；
- `@Resource @Inject`功能不如`@Autowired`，若无必要无需过分关注这两个注解。

### Bean作用范围及生命过程方法

- Bean配置默认的作用范围时`singleton`；
- 可以通过`@Scope`注解显示指定Bean的作用范围：

``` bash
@Scope("prototype")
@Component
public class MyComponent {
	@Autowired(required=false)
	private List<Plugin> plugins;
}
```
- 还可以通过`@PostConstruct`和`@PreDestroy`注解指定Bean的初始化及容器销毁前执行的方法：

``` bash
@Scope("prototype")
@Component
public class MyComponent {
	@Autowired(required=false)
	private List<Plugin> plugins;

	@PostConstruct
	private void init() {
		...
	}

	@PreDestroy
	private void destroy() {
		...
	}
}
```
- Spring先调用构造函数实例化Bean，再执行`@Autowired`进行自动注入，再分别执行`@PostConstruct`方法，当容器关闭时，分别执行`@PreDestroy`方法。

## 基于Java类的配置

### 使用Java类提供Bean定义信息

- 普通POJO(简单的Java对象，实际就是普通JavaBeans，是为了避免和EJB混淆所创造的简称)只要标注`@Configuration`，就可以为容器提供Bean的定义信息，每个标注了`@Bean`的类方法都相当于提供一个Bean的定义信息：

``` bash
@Configuration
public class MyApp {
	@Bean
	public UserDao userDao(){
		...
	}
	@Bean
	public MyService appService() {
		MyService myService = new MyService();
		myService.setUserDao(userDao());
		return myService
	}
}
```
- 以上的配置和一下XML配置等效：

``` bash
<bean id="userDao" class="com.lql.util.UserDao"/>
<bean id="myService" class="com.lql.util.MyService"
	p:userDao-ref="userDao"/>
```

### 使用基于Java类的配置信息启动Spring容器
- 通过标注`@Configuration`的Java类启动Spring容器;
- 还支持通过编码的方式加载多个`@Configuration`配置类，然后通过刷新容器应用这些配置类：

``` bash
@
AnnotatinConfigApplicationContext ctx = new AnnotatinConfigApplicationContext(MyService.class);
ctx.register(DaoConfig.class);
ctx.refresh();
MyApp myApp  = ctx.getBean(MyService.class);
...
```

- 也可以通过`@Import`将多个配置类组装到一个配置类中，这样仅需要注册这个组装好的配置类就可以启动容器了：

``` bash
@Configuration
@import(DaoConfig.class)
public class MyDao {
	...
}
```

- 如果希望将配置类组装到XML配置文件中，通过XML配置文件启动Spring容器，仅需要在XML中通过`<context:component-scan>扫描到相应的配置类就可以了：

``` bash
<context:component-scan base-package="com.lql.conf"
	resource-pattern="AppConf.class"/>
```

- 如果在beans.xml中定义了两个Bean，在`@Configuration`配置类中可通过`@ImportResource`引入XML配置文件，在配置类中即可通过`@Autowired`实现XML配置中Bean的注入：

``` bash
@Configuration
@ImportResource("classpath:com/lql/conf/beans.xml")
public class AppConfig {
	@Bean
	@AutoWired
	public MyService myService(...) {
		...
	}
}
```

## 不同配置方式取舍

基于XML配置 | 基于注解配置 | 基于Java类配置
----|------|----
1.Bean实现类来源于第三方类库，如DataSource,JdbcTemplate、SessionFactory等，因无法在类中标注注解，通过XML配置方式较好；2.命名空间的配置，如aop、context等，只能采用基于XML的配置。 | Bean的实现类是当前项目开发的，可以直接在Java类中使用基于注解的配置。  | 基于Java配置的优势在于可以通过代码方式控制Bean初始化的整体逻辑，所以如果实例化Bean的逻辑比较负复杂，则比较适用基于Java类配置的方式。

