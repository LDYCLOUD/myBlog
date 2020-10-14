---
title: Spring-Aop源码
date: 2020-10-14 13:58:30
tags:
- java
- spring
categories:
---

### Spring-Aop源码

#### Aop Demo

配置类：AopConfig.java

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan("com.ldy.ldystudent.aop")
public class AopConfig {

    @Bean
    public AopInterface aopInterface(){
        return new LdyAop() ;
    }

}
```

<!--more-->

接口：AopInterface.java

```java
public interface AopInterface {

    void test01() ;

    void test02() ;

    void test03() ;

}
```

实现类：LdyAop.java

```java
@Component
public class LdyAop implements AopInterface {
    @Override
    public void test01() {
        System.out.println("hello test01") ;
    }

    @Override
    public void test02() {
        System.out.println("hello test02") ;
    }

    @Override
    public void test03() {
        System.out.println("hello test03") ;
    }
}
```

增强类：LdyAspect.java

```java
@Aspect
@Order
@Component
public class LdyAspect {

//    /*引入:*/
//    @DeclareParents(value="com.ldy.ldystudent.aop.LdyAop",   // 动态实现的类
//            defaultImpl = SimpleProgramCalculate.class)  // 引入的接口的默认实现
//    public static ProgramCalculate programCalculate;    // 引入的接口

    @Pointcut("execution(* com.ldy.ldystudent.aop.LdyAop.*(..))")
//    @Pointcut("execution(* com.ldy.ldystudent.aop.LdyAop.test01())")
    public void pointcut(){} ;

    @Before(value = "pointcut()")
    public void before(JoinPoint joinPoint){
        System.out.println(joinPoint.getSignature().getName()+"的前置方法") ;
    }

    @After(value = "pointcut()")
    public void after(JoinPoint joinPoint){
        System.out.println(joinPoint.getSignature().getName()+"的后置方法") ;
    }

    @AfterReturning(value = "pointcut()")
    public void afterReturning(JoinPoint joinPoint){
        System.out.println(joinPoint.getSignature().getName()+"的后置返回方法") ;
    }

    @AfterThrowing(value = "pointcut()")
    public void methodAfterThrowing(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName()+"的一场通知") ;
    }
}
```

执行类：LdyMain.java

```java
public class LdyMain {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(AopConfig.class) ;
        AopInterface ldyAop = (AopInterface) annotationConfigApplicationContext.getBean("ldyAop");
        ldyAop.test02();
    }

}
```

执行结果：

```java
14:12:59.590 [main] DEBUG org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory - Found AspectJ method: public void com.ldy.ldystudent.aop.LdyAspect.methodAfterThrowing(org.aspectj.lang.JoinPoint)
14:12:59.776 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'ldyAop'
14:12:59.830 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'ldyAspect'
14:12:59.835 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'aopInterface'
test02的前置方法
hello test02
test02的后置返回方法
test02的后置方法
Disconnected from the target VM, address: '127.0.0.1:0', transport: 'socket'
```

以上就完成了简单的Aop类的增强。

#### 源码解析

我们知道，spring中的aop是通过动态代理实现的，那么他具体是如何实现的呢?spring通过一个切面类，在他的类上加入@Aspect注 解，定义一个Pointcut方法，最后定义一系列的增强方法。这样就完成一个对象的切面操作。 那么思考一下，按照上述的基础，要实现我们的aop，大致有以下思路:
 1.找到所有的切面类

2.解析出所有的advice并保存 

3.创建一个动态代理类

4.调用被代理类的方法时，找到他的所有增强器，并增强当前的方法

##### 一、切面类解析

spring通过@EnableAspectJAutoProxy开启aop切面，在注解类上面发现@Import(AspectJAutoProxyRegistrar.class)， AspectJAutoProxyRegistrar实现了ImportBeanDefinitionRegistrar，所以他会通过registerBeanDefinitions方法为我们容器导入 beanDefinition。

![截屏2020-10-14 14.22.41](./1.1.png)

###### 进入解析切面的过程:

postProcessBeforeInstantiation是在任意bean创建的时候就调用了 org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantia org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessBeforeInstantiation

![截屏2020-10-14 14.48.34](./1.2.png)

追踪一下源码可以看到最终导入AnnotationAwareAspectJAutoProxyCreator，我们看一下他的类继承关系图，发现它实现了两个重要 的接口，BeanPostProcessor和InstantiationAwareBeanPostProcessor 首先看InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法

Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName)(InstantiationAwareBeanPostProcessor)

org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessBeforeInstantiation org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator#shouldSkip org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

```java
/**
 * Look for AspectJ-annotated aspect beans in the current bean factory,
 * and return to a list of Spring AOP Advisors representing them.
 * <p>Creates a Spring Advisor for each AspectJ advice method.
 * @return the list of {@link org.springframework.aop.Advisor} beans
 * @see #isEligibleBean
 */
public List<Advisor> buildAspectJAdvisors() {
  // 获取缓存中的aspectBeanNames
   List<String> aspectNames = this.aspectBeanNames;

   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
           // 获取beanFactory中所有beanNames
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) {
               if (!isEligibleBean(beanName)) {
                  continue;
               }
               // We must be careful not to instantiate beans eagerly as in this case they
               // would be cached by the Spring container but would not have been weaved.
               Class<?> beanType = this.beanFactory.getType(beanName, false);
               if (beanType == null) {
                  continue;
               }
              //找出所有类上面含@Aspect注解的beanName
               if (this.advisorFactory.isAspect(beanType)) {
                 //将找到的beanName放入aspectNames集合
                  aspectNames.add(beanName);
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                    //1.找到切面类的所有但是不包括@Pointcut注解的方法
										//2.筛选出来包含@Around, @Before, @After,@ AfterReturning， @AfterThrowing注解的方法 
                    //3.封装为List<Advisor>返回
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                     if (this.beanFactory.isSingleton(beanName)) {
                       //将上面找出来的Advisor按照key为beanName，value为List<Advisor>的形式存入advisorsCache
                        this.advisorsCache.put(beanName, classAdvisors);
                     }
                     else {
                        this.aspectFactoryCache.put(beanName, factory);
                     }
                     advisors.addAll(classAdvisors);
                  }
                  else {
                     // Per target or per this.
                     if (this.beanFactory.isSingleton(beanName)) {
                        throw new IllegalArgumentException("Bean with name '" + beanName +
                              "' is a singleton, but aspect instantiation model is not singleton");
                     }
                     MetadataAwareAspectInstanceFactory factory =
                           new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                     this.aspectFactoryCache.put(beanName, factory);
                     advisors.addAll(this.advisorFactory.getAdvisors(factory));
                  }
               }
            }
            this.aspectBeanNames = aspectNames;
            return advisors;
         }
      }
   }

   if (aspectNames.isEmpty()) {
      return Collections.emptyList();
   }
   List<Advisor> advisors = new ArrayList<>();
   for (String aspectName : aspectNames) {
     //当再次进入该方法，会直接从advisorsCache缓存中获取
      List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
      if (cachedAdvisors != null) {
         advisors.addAll(cachedAdvisors);
      }
      else {
         MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
         advisors.addAll(this.advisorFactory.getAdvisors(factory));
      }
   }
   return advisors;
}
```

##### 二、创建代理

postProcessAfterInitialization是在bean创建完成之后执行的 org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitializati org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization

1.获取advisors:创建代理之前首先要判断当前bean是否满足被代理， 所以需要将advisor从之前的缓存中拿出来和当前bean 根据表达式进行匹配:

Object postProcessAfterInitialization(@Nullable Object bean, String beanName)(BeanPostProcessor) org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

上述代码的链路最终到了findCandidateAdvisors，我们发现在postProcessBeforeInstantiation方法中对查找到的Advisors做了缓存， 所以这里只需要从缓存中取就好了
 最后创建代理类，并将Advisors赋予代理类，缓存当前的代理类

2.匹配:根据advisors和当前的bean根据切点表达式进行匹配，看是否符合。

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply org.springframework.aop.support.AopUtils#findAdvisorsThatCanApply org.springframework.aop.support.AopUtils#canApply(org.springframework.aop.Advisor, java.lang.Class<?>, boolean)

拿到PointCut

org.springframework.aop.support.AopUtils#canApply(org.springframework.aop.Pointcut, java.lang.Class<?>, boolean) org.springframework.aop.ClassFilter#matches 粗筛

org.springframework.aop.IntroductionAwareMethodMatcher#matches 精筛

3.创建代理:找到了 和当前Bean匹配的advisor说明满足创建动态代理的条件:

```java
Object proxy = createProxy(
      bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
```

调用ProxyFactory创建动态代理

这里创建会选择JDK和CGLIB 两大因素

有接口：JDK代理

无接口：CGLIB代理

理解了上面两个重要的方法，我们只需要将他与创建bean的流程联系起来就可以知道代理对象创建的整个流程了，在before和after方法 分别放置断点，我们可以看到他的整个调用链路

##### 三、代理类的调用

前面的分析可知，spring将找到的增强器Advisors赋予了代理类，那么在执行只要将这些增强器应用到被代理的类上面就可以了，那么 spring具体是怎么实现的呢，下面我们以jdk代理为例分析一下源码:

org.springframework.aop.framework.JdkDynamicAopProxy#invoke

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   Object oldProxy = null;
   boolean setProxyContext = false;
	
    //获取当前被代理类
   TargetSource targetSource = this.advised.targetSource;
   Object target = null;
		// equals，hashcode等方法不做代理，直接调用
   try {
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         return equals(args[0]);
      }
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) {
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }

      Object retVal;
			// 将代理对象放到线程本地变量中
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }

      // Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      target = targetSource.getTarget();
      Class<?> targetClass = (target != null ? target.getClass() : null);

      // Get the interception chain for this method.
     //将增加器装换为方法执行拦截器链
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
        //将拦截器链包装为ReflectiveMethodInvocation并执行
         MethodInvocation invocation =
               new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         retVal = proxy;
      }
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
   }
   finally {
      if (target != null && !targetSource.isStatic()) {
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

通过上面代码可知，将增强器装换为方法拦截器链，最终包装为ReflectiveMethodInvocation执行它的proceed方法，那么我们就来看下 具体如果执行

```java
@Override
@Nullable
public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early.
  // 当执行到最后一个拦截器的时候才会进入
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }

   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
      if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed();
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      // 执行拦截器方法
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

这样一看会感觉很蒙，其实追踪一下源码就很好理解了 

org.springframework.aop.interceptor.ExposeInvocationInterceptor#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   MethodInvocation oldInvocation = invocation.get();
   invocation.set(mi);
   try {
      return mi.proceed();
   }
   finally {
      invocation.set(oldInvocation);
   }
}
```

org.springframework.aop.aspectj.AspectJAfterThrowingAdvice#invoke

异常拦截器，当方法调用异常会被执行

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   catch (Throwable ex) {
      if (shouldInvokeOnThrowing(ex)) {
         invokeAdviceMethod(getJoinPointMatch(), null, ex);
      }
      throw ex;
   }
}
```

org.springframework.aop.framework.adapter.AfterReturningAdviceInterceptor#invoke 

返回拦截器，方法执行失败，不会调用

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   Object retVal = mi.proceed();
   this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
   return retVal;
}
```

org.springframework.aop.aspectj.AspectJAfterAdvice#invoke 

后置拦截器，总是执行

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   finally {
      invokeAdviceMethod(getJoinPointMatch(), null, null);
   }
}
```

org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor#invoke 

前置拦截器

```java
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {

      methodBeforeAdvice.before(invocation.getMethod(),invocation.getArguments(),invocation.getThis());
    return invocation.proceed();
}
```

这里用了责任链的设计模式，递归调用排序好的拦截器链.