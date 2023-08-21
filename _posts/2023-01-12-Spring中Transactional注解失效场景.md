---
title: Spring中Transactional注解失效场景
categories: [Spring,Java]
comments: true
---

@Transactional 的原理是通过动态代理增强方法，形成 try catch 结构，捕获异常并进行回滚。
```java
public class A{
	@Transactional
	public void method(){
	
	}
}

public class ProxyA{
	public void method(){
		try{
			setAutoCommit(false);
			// 方法
			A.method();
			commit;
		}catch(e){
			rollback;
		}	
	}
}
```

![](/assets/img/Spring中Transactional注解失效场景/@Transactional.png)

### 一、事务不生效【七种】
#### 1.访问权限(只有 public 方法会生效)
Spring 事务管理器默认对 public 修饰的方法生效，可以通过设置为生效，但是不推荐

```java
@Bean
public TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource(false);
}
```

在 `AbstractFallbackTransactionAttributeSource` 类的 `computeTransactionAttribute` 方法中有个判断，如果目标方法不是 `public`，则 `TransactionAttribute` 返回null，即不支持事务。

#### 2.方法用final修饰，不会生效
如果某个方法用final修饰了，那么在它的代理类中，就无法重写该方法，而添加事务功能。
注意：如果某个方法是 static 的，同样无法通过动态代理，变成事务方法。

#### 3.同一个类中的方法直接内部调用，会导致事务失效
@Transactional 底层是动态代理，调用时调用的是动态代理类的方法，而内部直接调用相当于 this.method()，调用的并不是动态代理类增强后的方法。

解决方法：
1. 依赖注入自己（循环依赖，注入的是动态代理类）
2. 通过 `AopContext.currentProxy()` 获取动态代理类，需要 `@EnableAspectJAutoProxy(exposeProxy = true)` 开启
3. 通过 `CTW`，`LTW` 实现功能增强

#### 4.(类本身) 未被spring管理
Spring 的各种功能（@Autowired、@Resource、……）只能作用在 Spring 容器里的对象。

#### 5.多线程调用
```java
@Transactional
public void add(){
	new Thread(() -> {
		userService.doSomething();
	}).start();
}
```

两个方法不在同一个线程中，获取到的数据库连接不一样，从而是两个不同的事务。
spring的事务是通过数据库连接来实现的。当前线程中保存了一个map，key是数据源，value是数据库连接。
```java
private static final ThreadLocal<Map<Object, Object>> resources =
  new NamedThreadLocal<>("Transactional resources");
```

我们说的同一个事务，其实是指同一个数据库连接，只有拥有同一个数据库连接才能同时提交和回滚。如果在不同的线程，拿到的数据库连接肯定是不一样的，所以是不同的事务。

<font color=red>注意：多线程保证原子性时 synchronized 与 @Transactional 各自的范围。</font>

#### 6.(存储引擎)表不支持事务
MyISAM 不支持事务，但是有些老项目中，可能还在用它。

#### 7.未开启事务
当有父子容器时，需要注意扫描的包。比如：父容器配置类中开启了事务，子容器没有配置，而父子容器都扫描了 service 包，此时依赖注入时优先找子容器，找到的是没有事务管理的类。

### 二、事务不回滚【五种】
#### 1.错误的传播特性
我们在使用 `@Transactional` 注解时，是可以指定 `propagation` 参数的。

该参数的作用是指定事务的传播特性，spring目前支持7种传播特性：
- `REQUIRED`：如果当前上下文中存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值。  
- `SUPPORTS`：如果当前上下文存在事务，则支持事务加入事务，如果不存在事务，则使用非事务的方式执行。  
- `MANDATORY`：如果当前上下文中存在事务，否则抛出异常。  
- `REQUIRES_NEW`：每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行。  
- `NOT_SUPPORTED`：如果当前上下文中存在事务，则挂起当前事务，然后新的方法在没有事务的环境中执行。  
- `NEVER`：如果当前上下文中存在事务，则抛出异常，否则在无事务环境上执行代码。  
- `NESTED`：如果当前上下文中存在事务，则嵌套事务执行，如果不存在事务，则新建事务。

#### 2.手动捕获了异常
```java
@Slf4j
@Service
public class UserService {
    @Transactional
    public void add(UserModel userModel) {
        try {
            saveData(userModel);
            updateData(userModel);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
}
```

如果想要spring事务能够正常回滚，必须抛出它能够处理的异常。如果没有抛异常，则spring认为程序是正常的。

#### 3.手动抛了别的异常
Spring 事务，默认情况下只会回滚 RuntimeException（运行时异常）和 Error（错误），对于普通的 Exception（[非运行时异常](https://blog.csdn.net/mccand1234/article/details/51579425?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522165164871416780357228953%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=165164871416780357228953&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-51579425.nonecase&utm_term=error&spm=1018.2226.3001.4450)），它不会回滚。比如常见的 IOExeption 和 SQLException。

#### 4.自定义了回滚异常
在使用 @Transactional 注解声明事务时，有时我们想自定义回滚的异常，spring也是支持的。可以通过设置 `rollbackFor` 参数，来完成这个功能。
如果使用默认值，一旦程序抛出了 Exception，事务不会回滚，这会出现很大的 bug。所以，建议一般情况下，将该参数设置成：Exception 或 Throwable。

#### 5.嵌套事务回滚多了
```java
public class UserService {
    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RoleService roleService;
    
    @Transactional
    public void add(UserModel userModel) throws Exception {
        userMapper.insertUser(userModel);
        roleService.doOtherThing();
    }
}

@Service
public class RoleService {
    @Transactional(propagation = Propagation.NESTED)
    public void doOtherThing() {
        System.out.println("保存role表数据");
    }
}
```

嵌套事务会向外抛出异常，原本是希望调用 `roleService.doOtherThing` 方法时，如果出现了异常，`只回滚doOtherThing` 方法里的内容，不回滚 `userMapper.insertUser` 里的内容，即回滚保存点。但事实是，`insertUser` 也回滚了。

因为 `doOtherThing` 方法出现了异常，没有手动捕获，会继续往上抛，到外层 `add` 方法的代理方法中捕获了异常。所以，这种情况是直接回滚了整个事务，不只回滚单个保存点。