---
title: Druid读取jdbc.properties文件初始化失败问题
date: 2018-09-05 
tags: Java,Druid
excerpt_separator: <!--more-->
---
## 背景
在配置测试项目的数据库连接池时，我选用了alibaba的开源数据库连接池Druid，毕竟官方在项目的[wiki](https://github.com/alibaba/druid/wiki/%E9%A6%96%E9%A1%B5)中宣称——**DruidDataSource是最好的数据库连接池**，扛得住双十一千万级并发量的，说起话来就是有底气。  
btw，吐槽一下，在[英文版的wiki](https://github.com/alibaba/druid/wiki/FAQ)中，这句话就变成了`之一`：Druid is one of the best database connection pools written in JAVA。是因为加上了`in JAVA`这个定语吗？:joy:  
<!--more-->

## 配置
连接池的配置我参考了官方推荐的一个配置，因为我使用的是MySQL，关闭了PScashe，Druid会根据`url`的配置自动识别DriverClass，所以也无需配置了，在properties的配置文件中我也没有写上。
```Java
package com.alibaba.druid.pool;

if (this.driver == null) {
    if (this.driverClass == null || this.driverClass.isEmpty()) {
        this.driverClass = JdbcUtils.getDriverClassName(this.jdbcUrl);
}
```
```Java
package com.alibaba.druid.util;

else if (rawUrl.startsWith("jdbc:mysql:")) {
	if (mysql_driver_version_6 == null) {
		mysql_driver_version_6 = Utils.loadClass("com.mysql.cj.jdbc.Driver") != null;
	}

	return mysql_driver_version_6 ? "com.mysql.cj.jdbc.Driver" : "com.mysql.jdbc.Driver";
} 
```
`jdbc.properties`文件内容，仅配置了简单的连接属性。
```properties
url=jdbc:mysql://localhost:3306/seckill?characterEncoding=utf8
username=root
password=admin
```
## 调试
在做DAO的单元测试时（详细的DAO与单元测试类可见这个[仓库](https://github.com/Kaka2y/seckill_realize)），遇到了如下的错误。报错的信息是说Druid初始化错误，这个`url:jdbc:mysql://localhost:3306/seckill?useUnicode=true&characterEncoding=utf8` 是错误的。
```
21:32:11.664 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory - Invoking init method  'init' on bean with name 'dataSource'
21:32:12.529 [main] ERROR c.alibaba.druid.pool.DruidDataSource - init datasource error, url: jdbc:mysql://localhost:3306/seckill?useUnicode=true&characterEncoding=utf8
```
难道就因为我不写DriverClass吗？:joy:经过一番补全之后，测试结果依旧是这个错误。  

秉着不信任的态度，我把数据库的连接属性直接写在了Druid的bean中，而不使用Spring提供的占位符方式，又进行了一次测试。测试通过，说明我的连接属性字段配置是正确的。那么可以推测是读取配置文件出错。  
下面假设了可能出现的问题：
- 读取不到jdbc.properties配置文件
- 读取不到对应的连接字符串
- 读取到错误的连接字符串

检查了配置文件的加载，发现并没有错误，所以尝试用断点调试来查看Druid调用init()方法初始化过程中的变量值。  

经过查找，在DruidDataSource.java类中设置断点，这个`try catch`会抛出之前出现在控制台的异常字段。
```Java
try {
	i = 0;

	while(true) {
		if (i >= this.initialSize) {
			if (this.poolingCount > 0) {
				this.poolingPeak = this.poolingCount;
				this.poolingPeakTime = System.currentTimeMillis();
			}
			break;
		}
		//断点
		PhysicalConnectionInfo pyConnectInfo = this.createPhysicalConnection();
		DruidConnectionHolder holder = new DruidConnectionHolder(this, pyConnectInfo);
		this.connections[this.poolingCount] = holder;
		this.incrementPoolingCount();
		++i;
	}
} catch (SQLException var18) {
	LOG.error("init datasource error, url: " + this.getUrl(), var18);
	connectError = var18;
}
```
调试得到以下的结果：

![图片](http://t1.aixinxi.net/o_1cmp1g5b41km118hajdn1ju66qea.png-j.jpg)

可以发现`username`对应的值由定义的`root`变成了`13326`，后者是我的计算机的用户名，所以推断是Druid读取的数据出错了。

## 解决方法
可是为什么会出错呢？我在`DruidDataSource.java`中找到了如下内容：
```Java
var3 = properties.entrySet().iterator();

while(var3.hasNext()) {
	Entry entry = (Entry)var3.next();
	Object value = this.connectProperties.get(entry.getKey());
	Object entryValue = entry.getValue();
	if (value == null && entryValue != null) {
		equals = false;
		break;
	}
	//在本程序中执行了这个判断逻辑
	if (!value.equals(entry.getValue())) {
		equals = false;
		break;
	}
}
```
在`setConnectProperties()`方法中，通过Key来获取到的值和直接getValue得到的值是不一致的。
>Object value = this.connectProperties.get(entry.getKey());
Object entryValue = entry.getValue();

Key值为`username`时会直接从系统的配置文件中获取到本机的username，所以在连接数据库时，系统的配置文件用户属性的优先级优于数据库的用户属性，导致读取出错。  

那么只能改以下jdbc中的配置字段名称了，参考Druid官方给出的占位符做出修改：  
```properties
jdbc_url=jdbc:mysql://localhost:3306/seckill?characterEncoding=utf8
jdbc_user=root
jdbc_password=admin
```
下面是新的断点调试的结果：

![结果](http://t1.aixinxi.net/o_1cmp1hrv52ho12pi18eh1elql3fa.png-j.jpg)

控制台的执行日志正常：
```
22:01:58.024 [main] DEBUG org.seckill.dao.SeckillDao.queryById - <==      Total: 1
```
## The End
今天重新搜索问题时，发现在Druid项目下已经有人在之前提了这个[issue](https://github.com/alibaba/druid/issues/1415)，这个用户jiangmitiao做出的回答也被Druid的项目负责人采纳了。
>1.0.15版本，发现同样问题。
应该是windows下，username已经是一个property，优先级比自定义的高。spring先获得用户自定义的username，接着windows的property覆盖了用户的username。

