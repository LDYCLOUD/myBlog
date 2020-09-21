---
title: Spring-IOC源码
date: 2020-09-21 15:20:38
tags:
- java
- spring
categories:
---

### Spring-IOC加载过程

#### 前言

Spring 最重要的概念是 IOC 和 AOP，其中IOC又是Spring中的根基

#### 1.实例化容器：AnnotationConfigApplicationContext

从这里出发：

```java
// 加载spring上下文
ApplicationContext context=new AnnotationConfigApplicationContext(Start.class);
```

<!--more-->
可以通过idea工具查看AnnotationConfigApplicationContext类的结构关系

![截屏2020-09-21 16.06.13](./1-1.png)

创建AnnotationConfigApplicationContext对象

![截屏2020-09-21 16.08.54](./1-2.png)

简单说明：

1. 这是一个有参的构造方法，可以接收多个配置类，不过一般情况下，只会传入一个配置类。
2.  这个配置类有两种情况，一种是传统意义上的带上@Configuration注解的配置类，还有一种是没有 带上@Configuration，但是带有@Component，@Import，@ImportResouce，@Service， @ComponentScan等注解的配置类。

##### 1.1 先调用this()

如下代码：

```java
/**
 * Create a new AnnotationConfigApplicationContext that needs to be populated
 * through {@link #register} calls and then manually {@linkplain #refresh refreshed}.
 */
public AnnotationConfigApplicationContext() {
   this.reader = new AnnotatedBeanDefinitionReader(this);
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

此时会创建两个类：

​	this.reader = new AnnotatedBeanDefinitionReader(this)：注解的Bean定义读取器。

（解析：internalConfigurationAnnotationProcessor，internalAutowiredAnnotationProcessor，internalCommonAnnotationProcessor等里面的配置类）

 this.scanner = new ClassPathBeanDefinitionScanner(this)：它仅仅是在外面手动调用.scan方法，或者调用参数为String的构造方法，传 入需要扫描的包名才会用到。

#### 2.实例化工厂:DefaultListableBeanFactory

#### 3.实例化建BeanDefinition读取器: AnnotatedBeanDefinitionReader

#### 4.创建BeanDefinition扫描器:ClassPathBeanDefinitionScanner

#### 5.注册配置类为BeanDefinition: register(annotatedClasses)

#### 6.refresh()