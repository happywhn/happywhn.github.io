---
title: Spring 配置数据源时 errorCode 1045, state 28000
categories: [Java,Spring]
comments: true
---
- [错误原因](#错误原因)
- [解决方法](#解决方法)

### 错误原因

Spring 在配置数据源时报错

<font color=red>ERROR 09-01 15:02:05,294 create connection SQLException, url: jdbc:mysql://localhost:3306/voting?useSSL=false, errorCode 1045, state 28000  (DruidDataSource.java:2840) 

java.sql.SQLException: Access denied for user 'whn'@'localhost' (using password: YES)</font>

![](/assets/img/Spring配置数据源errorCode-1045-state-28000/异常信息.png)

我们看错误内容，拒绝访问 'whn'@'localhost'，可以看到这里登录使用的用户是 whn，这里 whn 是我计算机的用户名

properties
```properties
driverClassName=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost:3306/voting?useSSL=false
user=root
pwd=root
maxActive=50
maxIdle=20
maxWait=60000
```

从 jdbc.properties 文件中我们可以看出登录使用的用户是 root

idea 中显示的内容

![](/assets/img/Spring配置数据源errorCode-1045-state-28000/idea显示数据.png)

可以看到配置文件成功加载，并解析了内容

**配置文件中的 username 不能叫 username，spring 会默认 username 是你的计算机用户名。**

### 解决方法

在编写 properteis 配置文件时加上前缀，可以避免遇到名字相同的情况

```properties
jdbc.driverClassName=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/voting?useSSL=false
jdbc.user=root
jdbc.pwd=root
jdbc.maxActive=50
jdbc.maxIdle=20
jdbc.maxWait=60000
```
