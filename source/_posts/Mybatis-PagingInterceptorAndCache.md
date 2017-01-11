---
title: Mybatis相爱相杀-分页拦截器及相关的缓存机制
date: 2017-01-03 11:48:53
categories: [Mybatis,Java]
tags: [Mybatis,分页,拦截器,缓存]
---

> 数据库分页是项目中经常用到的技术，在这里总结一下自己使用的方法及遇到的问题总结

<!-- more -->

## 原理

### 分页目的

- 分页,是一种将所有数据分段展示给用户的技术.用户每次看到的不是全部数据,而是其中的一部分,如果在其中没有找到自己想要的内容,用户可以通过制定页码或是翻页的方式转换可见内容,直到找到自己想要的内容为止。
- 减少数据库查询的压力，加快查询的速度，让客户尽可能快地看到想要的有效数据（控制输出结果集大小，将结果尽快的返回）；
- 避免客户看到繁多的数据而难以便利地筛选想要的数据。
- 缺点:造成了系统设计的复杂度。

### 分页点

- 分页可在浏览器、服务器、数据库端进行；
	- ![web数据交互](/images/web数据交互.gif)
- 选择的标准在于速度；
- 服务器与客户端靠网络连接，为了加快应用端响应速度，应避免传输大量的数据，而且客户端处理数据的能力有限，所以客户端的浏览器分页不可取；
- 如果选择在Web服务器端分页的话，大部分的被过滤掉的数据还是被传输到了Web应用服务器端，而且加大了在服务器端的存储，所以采取数据库的分页；
- 比较好的分页做法应该是每次翻页的时候只从数据库里检索页面大小的块区的数据。


### 分页的SQL语句

- MySql的Limit m,n语句
	- Limit后的两个参数中，参数m是起始下标，它从0开始；参数n是返回的记录数。我们需要分页的话指定这两个值即可
- Oracle数据库的rownum
	- Rownum表示一条记录的行号,值得注意的是它在获取每一行后才赋予。因此，想指定rownum的区间来取得分页数据在一层查询语句中是无法做到的，要分页还要进行一次查询。
	``` bash
	SELECT * FROM(
		SELECT A.*, ROWNUM RN
		FROM (SELECT * FROM TABLE_NAME) A
			WHERE ROWNUM <= 40)
	WHERE RN >= 21
	```

### 数据库分页查询原理

- 利用JDBC对数据库进行操作就必须要有一个对应的Statement对象，Mybatis在执行Sql语句前也会产生一个包含Sql语句的Statement对象，而且对应的Sql语句是在Statement之前产生的，所以就可以在它生成Statement之前对用来生成Statement的Sql语句下手。在Mybatis中Statement语句是通过RoutingStatementHandler对象的prepare方法生成的。所以利用拦截器实现Mybatis分页的一个思路就是拦截StatementHandler接口的prepare方法，然后在拦截器方法中把Sql语句改成对应的分页查询Sql语句，之后再调用StatementHandler对象的prepare方法，即调用invocation.proceed()。

## Mybatis拦截器

### 分页操作封装

- 对于分页而言，在拦截器里面我们常常还需要做的一个操作就是统计满足当前条件的记录一共有多少，这是通过获取到了原始的Sql语句后，把它改为对应的统计语句再利用Mybatis封装好的参数和设置参数的功能把Sql语句中的参数进行替换，之后再执行查询记录数的Sql语句进行总记录数的统计。先来看一个我们对分页操作封装的一个实体类Pager：

``` bash
import java.util.ArrayList;
import java.util.List;
public class Pager<T> {
	/**
	 * 分页的大小
	 */
	private int limit;
	/**
	 * 分页的起始位置
	 */
	private int start;
	/**
	 * 当前页码
	 */
	private int page;
	/**
	 * 总记录数
	 */
	private long total;
	/**
	 * 分页的数据
	 */
	private List<T> datas= new ArrayList<T>();

    public int getLimit() {
        return limit;
    }

    public void setLimit(int limit) {
        this.limit = limit;
    }

    public int getPage() {
        return page;
    }

    public void setPage(int page) {
        this.page = page;
    }

    public int getStart() {
        return start;
    }

    public void setStart(int start) {
        this.start = start;
    }

    public long getTotal() {
		return total;
	}
	public void setTotal(long total) {
		this.total = total;
	}
	public List<T> getDatas() {
		return datas;
	}
	public void setDatas(List<T> datas) {
		this.datas = datas;
	}
}
```
- 对于需要进行分页的Mapper映射，我们会给它传一个Pager对象作为参数，我们可以看到Pager对象里面包括了一些分页的基本信息，这些信息我们可以在拦截器里面用到。

### 拦截器实现

- @Intercepts标记了这是一个Interceptor，然后在@Intercepts中定义了一个@Signature，即一个拦截点。定义了该Interceptor将拦截StatementHandler接口中参数类型为Connection、Integer的prepare方法；

- 该接口中一共定义有三个方法，intercept、plugin和setProperties。
	- plugin方法是拦截器用于封装目标对象的，通过该方法我们可以返回目标对象本身，也可以返回一个它的代理；
	- 当返回的是代理的时候我们可以对其中的方法进行拦截来调用intercept方法，当然也可以调用其他方法；
	- setProperties方法是用于在Mybatis配置文件中指定一些属性的。
- intercept和plugin是该接口中最重要的两个方法
	- 在plugin方法中我们可以决定是否要进行拦截进而决定要返回一个什么样的目标对象；
	- intercept方法就是要进行拦截的时候要执行的方法。

``` bash

import com.eking.emon.commons.model.SystemRequest;
import com.eking.emon.commons.model.SystemRequestHolder;
import com.eking.emon.commons.utils.ReflectUtil;
import com.eking.emon.commons.utils.StringUtil;
import org.apache.ibatis.executor.statement.BaseStatementHandler;
import org.apache.ibatis.executor.statement.RoutingStatementHandler;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.sql.Connection;
import java.util.Properties;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
@Intercepts({@Signature(method = "prepare", type = StatementHandler.class, args = {Connection.class, Integer.class})})
public class PageInterceptor implements Interceptor {

    private final static Logger logger = LoggerFactory.getLogger(PageInterceptor.class);

    private  String dialect = "";    //数据库方言
    private  String pageSqlId = ""; //mapper.xml中需要拦截的ID(正则匹配)

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if ((!(invocation.getTarget() instanceof RoutingStatementHandler))) {
            return invocation.proceed();
        }
        SystemRequest sr = SystemRequestHolder.getSystemRequest();
        if(sr==null){
            return invocation.proceed();
        }
        Integer offset = 0;
        Integer limit = 0;
        if(sr.getRequest().getParameter("start") != null){
            offset = Integer.parseInt(sr.getRequest().getParameter("start"));
        }
        if(sr.getRequest().getParameter("limit") != null){
            limit = Integer.parseInt(sr.getRequest().getParameter("limit"));
        }

        if (offset < 0 || limit <= 0) {
            return invocation.proceed();
        }

        RoutingStatementHandler statementHandler = (RoutingStatementHandler) invocation.getTarget();
        BaseStatementHandler delegate = (BaseStatementHandler) ReflectUtil.getFieldValue(statementHandler, "delegate");
        MappedStatement mappedStatement = (MappedStatement) ReflectUtil.getFieldValue(delegate, "mappedStatement");
        if (!mappedStatement.getId().matches(pageSqlId)) {
            return invocation.proceed();
        }
        BoundSql boundSql = statementHandler.getBoundSql();
        String sql = boundSql.getSql();
        // 重写sql
        String pageSql = getLimitString(sql, offset, limit);
        //强制修改最终要执行的SQL
        ReflectUtil.setFieldValue(boundSql, "sql", pageSql);
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }

    private String getLimitString(String sql, int offset, int limit) {
        if (offset < 0 || limit <= 0) {
            return "";
        }
        String trimedSql = sql.trim();
        if ("Mysql".equals(dialect)) {
            int orderByIndex = StringUtil.indexOfIgnoreCase(trimedSql, "ORDER BY");
            int whereByIndex = StringUtil.indexOfIgnoreCase(trimedSql, "WHERE");
            if (orderByIndex == -1) {
                logger.error(" 需分页的sql:" + trimedSql + ",缺少ORDER BY子句! ");
                return "";
            }
            Pattern pattern = Pattern.compile("\\(.*?\\sfrom\\s.*?\\)", Pattern.CASE_INSENSITIVE);
            Matcher matcher = pattern.matcher(trimedSql);
            StringBuffer sb = new StringBuffer();
            while (matcher.find()) {
                matcher.appendReplacement(sb, StringUtil.generateStr("#", matcher.group().length()));
            }
            matcher.appendTail(sb);
            int fromIndex = StringUtil.indexOfIgnoreCase(sb.toString(), "FROM");
            String selectStr = trimedSql.substring(0, fromIndex);
            String fromStr = trimedSql.substring(fromIndex, whereByIndex);
            String whereStr = trimedSql.substring(whereByIndex, orderByIndex);
            String orderByStr = trimedSql.substring(orderByIndex);
            StringBuilder sb2 = new StringBuilder("SELECT TT.* FROM (SELECT DD.*,@rownum := @rownum + 1 AS ROW_NUM FROM (").append(selectStr);
            sb2.append(fromStr).append(",(SELECT @rownum:=0) AS r ").append(whereStr).append(orderByStr);
            sb2.append(")DD )TT WHERE TT.ROW_NUM BETWEEN ").append(offset + 1).append(" AND ").append(offset + limit).append(" ORDER BY TT.ROW_NUM");
//            System.out.println("SQL = " + sb2.toString());
            return sb2.toString();
        }
        return trimedSql;
    }

    private boolean isSelect(String sql) {
        if (sql.trim().toUpperCase().startsWith("SELECT")) {
            return true;
        }
        return false;
    }

    public String getDialect() {
        return dialect;
    }

    public void setDialect(String dialect) {
        this.dialect = dialect;
    }

    public String getPageSqlId() {
        return pageSqlId;
    }

    public void setPageSqlId(String pageSqlId) {
        this.pageSqlId = pageSqlId;
    }

}
```

- Plugin的wrap方法，会根据注解上需要拦截的接口来判断拦截的对象是否实现对应需要拦截的接口，如果没有则返回目标对象本身，如果有则返回一个代理对象。
- 这个代理对象的InvocationHandler正是一个Plugin。所以当目标对象在执行接口方法时，如果是通过代理对象执行的，则会调用对应InvocationHandler的invoke方法，也就是Plugin的invoke方法。
- invoke方法的逻辑是：如果当前执行的方法是定义好的需要拦截的方法，则把目标对象、要执行的方法以及方法参数封装成一个Invocation对象，再把封装好的Invocation作为参数传递给当前拦截器的intercept方法。如果不需要拦截，则直接调用当前的方法。Invocation中定义了定义了一个proceed方法，其逻辑就是调用当前方法，所以如果在intercept中需要继续调用当前方法的话可以调用invocation的procced方法。
- Mybatis在注册该拦截器的时候就会利用定义好的n个property作为参数调用该拦截器的setProperties方法。


- 在Spring-config.xml中定义该拦截器：

``` bash
	<bean id="pageInterceptor" class="com.eking.emon.commons.peresist.PageInterceptor">
	        <property name="dialect" value="Mysql"/>
	        <property name="pageSqlId" value=".*select.*Page.*"/>
	</bean>
```

- 在通用的数据操作接口中定义分页查询

``` bash
	@Override
	public Pager<T> selectPage(Map<String,Object> paramMap,int totalSize,int pageSize) {
		Pager<T> resPage = new Pager<T>();
		//先获取记录总数
		resPage.setTotal(totalSize);
		SystemRequest sr = SystemRequestHolder.getSystemRequest();
		String start = sr.getRequest().getParameter("start");
		String limit = sr.getRequest().getParameter("limit");
		resPage.setStart(Integer.parseInt(start));
		resPage.setLimit(Integer.parseInt(limit));
		resPage.setDatas(super.getSqlSessionTemplate().<T>selectList(getQueryPageSql(), paramMap));
		System.out.println("total:" + totalSize + " pageSize:" + pageSize + " start:" + start + " limit:" + limit + " dataSize:" +resPage.getDatas().size());
		return resPage;
	}
	@Override
	public Pager<T> selectPage(Map<String, Object> paramMap, int pageSize) {
		int totalSize=countPage(paramMap);
		return selectPage(paramMap, totalSize, pageSize);
	}

	@Override
	public Pager<T> selectPage(Map<String,Object> paramMap) {
		return selectPage(paramMap, GlobalVars.PAGE_LIMIT);
	}
```

- 在service层及数据持久层中定义相应的page查询方法

## Mybatis缓存机制及出现问题

### 缓存结构
- 将数据缓存设计成两级结构，分为一级缓存、二级缓存：
	- 一级缓存是session会话级别的缓存，又称为本地缓存，位于表示一次数据库会话的SqlSession对象之中；其是内部实现的一个特性，用户不能配置，没有定制他的权力（不是绝对，可以通过开发插件对它进行修改）；
	- 二级缓存是Application应用级别的缓存，生命周期很长，跟Application的声明周期一样，作用范围是整个Application应用。
- 一级缓存与二级缓存的组织结构如下：
	-![一级缓存与二级缓存的组织结构](/images/一级缓存与二级缓存的组织结构.jpg)

### 工作机制
- 一级缓存工作机制：
	- 一级缓存是Session会话级别的，一般而言，一个SqlSession对象会使用一个Executor对象来完成会话操作，Executor对象会维护一个Cache缓存，以提高查询性能。
- 二级缓存工作机制：
	- 关键就是对这个用来完成回话操作的Executor对象做文章；
	- 如果用户配置了`cacheEnabled=true`，那么MyBatis在为SqlSession对象创建Executor对象时，会对Executor对象加上一个装饰者：`CachingExecutor`，这时SqlSession使用CachingExecutor对象来完成操作请求。
	- CachingExecutor对于查询请求，会先判断该查询请求在Application级别的二级缓存中是否有缓存结果，如果有查询结果，则直接返回缓存结果；如果缓存中没有，再交给真正的Executor对象来完成查询操作，之后CachingExecutor会将真正Executor返回的查询结果放置到缓存中，然后在返回给用户。

### 出现问题
- 问题：在进行相关的分页查询时，分页的显示总是出问题，总是显示第一页的数据。
- 原因：在开启cache时，executor在生成读取cache的key（key由sql以及对应的参数值构成）时使用都是原始的sql。
- 解决：在mybatis配置中，配置如下:

``` bash
<configuration>
	<settings>
		<setting name="cacheEnabled" value="false" />
	</settings>
</configuration>
```

## 参考 致谢
- [1.《深入理解mybatis原理》 MyBatis缓存机制的设计与实现](http://blog.csdn.net/luanlouis/article/details/41390801)
- [2. Mybatis拦截器介绍及分页插件](http://www.tuicool.com/articles/7bYjUn)