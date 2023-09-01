---
title: Spring Bean 循环依赖
categories: [Java,Spring]
comments: true
---
### 1 产生原因

![代码示例](/assets/img/Spring中Bean的循环依赖/循环依赖代码示例.png)

![循环依赖流程](/assets/img/Spring中Bean的循环依赖/循环依赖问题流程.png)

a 的创建需要 b 对象，但是 b 对象的创建有需要 a 对象，也就造成了循环依赖问题。

### 2 解决方案
#### 2.1 注入代理类
![循环依赖流程](/assets/img/Spring中Bean的循环依赖/解决方案-注入代理对象.png)

用 @Lazy 为构造方法参数生成代理
```java
public class LazyResolve {
    static class A {
        private static final Logger log = LoggerFactory.getLogger(A.class);
        private B b;
        public A(@Lazy B b) {
            log.debug("A(B b) {}",b.getClass());
            this.b = b;
        }
        @PostConstruct
        public void init(){
            log.debug("A init()");
        }
        // getter and setter
    }

    static class B {
        private static final Logger log = LoggerFactory.getLogger(B.class);
        private A a;

        public B(A a) {
            log.debug("B(A a) {}",a.getClass());
            this.a = a;
        }
        @PostConstruct
        public void init(){
            log.debug("B init()");
        }
        // getter and setter
    }

    public static void main(String[] args) {
        GenericApplicationContext context =
                new GenericApplicationContext();
        context.registerBean("a",A.class);
        context.registerBean("b",B.class);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.refresh();
        System.out.println("===================end===================");
    }
}
```

![Lazy注解测试控制台打印信息](/assets/img/Spring中Bean的循环依赖/Lazy注解测试控制台打印结果.png)


#### 2.2 延迟创建
- a 注入 b 的工厂对象，让 b 延迟创建，保证 a 的流程先走完
- 后续用到 b 时通过 ObjectFactory 工厂间接访问

![注入ObjectFactory创建过程](/assets/img/Spring中Bean的循环依赖/注入ObjectFactory创建过程.png)

用 ObjectProvider 延迟依赖对象的创建
```java
public class ObjectProviderResolve {
    static class A {
        private static final Logger log = LoggerFactory.getLogger(LazyResolve.A.class);
        private ObjectProvider<B> b;

        public A(ObjectProvider<B> b) {
            log.debug("A(B b) {}", b.getClass());
            this.b = b;
        }

        @PostConstruct
        public void init() {
            log.debug("A init()");
        }
        // getter and setter
    }

    static class B {
        private static final Logger log = LoggerFactory.getLogger(LazyResolve.B.class);
        private A a;

        public B(A a) {
            log.debug("B(A a) {}", a.getClass());
            this.a = a;
        }

        @PostConstruct
        public void init() {
            log.debug("B init()");
        }
        // getter and setter
    }

    public static void main(String[] args) {
        GenericApplicationContext context =
                new GenericApplicationContext();
        context.registerBean("a", A.class);
        context.registerBean("b", B.class);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.refresh();
        System.out.println(context.getBean(A.class).b.getObject());
        System.out.println(context.getBean(B.class));
    }
}
```

![ObjectProvider测试控制台打印结果](/assets/img/Spring中Bean的循环依赖/ObjectProvider测试控制台打印结果.png)

>还有 @Scope(proxyMode = ScopedProxyMode.TARGET_CLASS) 与 Provider 接口都可以解决。

### 3 Spring 三级缓存

我们可以通过查看 doGetBean 方法查看 Bean 的创建流程，以及其中三级缓存的作用。

#### 一级缓存(singletonObjects)

存放完全初始化好的 bean，再次获取时先从一级缓存中查找，保证单一实例

![一级缓存流程](/assets/img/Spring中Bean的循环依赖/一级缓存流程.png)


a 需要 b，b 又需要 a，无法解决循环依赖问题

#### 二级缓存(singletonFactories，Spring中叫三级缓存)


存放半成品 bean

![三级缓存流程](/assets/img/Spring中Bean的循环依赖/三级缓存流程.png)

对于上面的图
- a = new A() 后将这个半成品 a 放入 singletonFactories
- 接下来执行 a.setB()，进入 getBean("b")，红色箭头 3
- 这回再执行到 b.setA() 时，就能够从 singletonFactories 缓存中找到
- b 的流程能够顺利走完，将 b 成品放入 singletonObject 一级缓存，返回到 a
的依赖注入流程

目前为止，这两个缓存就已经解决循环依赖问题了，为什么还要再加入一级缓存？

- 因为无法处理包含代理的情况

![两个缓存中包含代理的场景](/assets/img/Spring中Bean的循环依赖/两个缓存中包含代理的场景.png)

- spring 默认要求，在 a.init 完成之后才能创建代理 pa = proxy(a)
- 由于 a 的代理创建时机靠后，在执行 factories.put(a) 向
singletonFactories 中放入的还是原始对象
- 接下来箭头 3、4、5 这几步 b 对象拿到和注入的都是原始对象

#### 三级缓存(earlySingletonObjects，Spring中的二级缓存)

![二级缓存流程](/assets/img/Spring中Bean的循环依赖/二级缓存流程.png)

一般这时候解决方案就是将代理创建提前，放在 init 前面，但是 spring 希望代理的创建时机在 init 之后，只有出现循环依赖时，才会将代理的创建时机提前。
所以解决思路稍显复杂：
- 图中 factories.put(fa) 放入的既不是原始对象，也不是代理对象而是工厂
对象 fa
- 当检查出发生循环依赖时，fa 的产品就是代理 pa，没有发生循环依赖，fa 的产品是原始对象 a
- 假设出现了循环依赖，拿到了 singletonFactories 中的工厂对象，通过在依赖注入前获得了 pa，红色箭头 5
- 这回 b.setA() 注入的就是代理对象，保证了正确性，红色箭头 7
- 还需要把 pa 存入新加的 earlySingletonObjects 缓存，红色箭头 6
- a.init 完成后，无需二次创建代理，从哪儿找到 pa 呢？earlySingletonObjects 已经缓存，蓝色箭头 9

当成品对象产生，放入 singletonObject 后，singletonFactories 和
earlySingletonObjects 就中的对象就没有用处，清除即可