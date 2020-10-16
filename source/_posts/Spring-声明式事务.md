---
title: Spring-声明式事务
date: 2020-10-16 09:22:20
tags:
- java
- spring
categories:
---

### Spring-声明式事务

#### @Transactional的使用

**SpringBoot**大行其道的今天，基于XML配置的Spring Framework的使用方式注定已成为过去式。 注解驱动应用，面向元数据编程已然成受到越来越多开发者的偏好了，毕竟它的便捷程度、优势都是XML方式 不可比拟的。

<!--more-->

##### 1、开启注解驱动

```java
@EnableTransactionManagement
@EnableAspectJAutoProxy(exposeProxy = true)
@ComponentScan(basePackages = {"com.ldy"})
public class MainConfig {


    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        dataSource.setUrl("jdbc:mysql://localhost:3306/ldy-ms-alibaba");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

		// 配置JdbcTemplate Bean组件
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

		// 配置事务管理器
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

}
```

**提示**:使用@EnableTransactionManagement注解前，请务必保证你已经配置了至少一个PlatformTransactionManager的Bean，否则会报错。(当 然你也可以实现TransactionManagementConfigurer来提供一个专属的，只是我们一般都不这么去做~~~)

##### 2、在你想要加入事务的方法上(或者类(接口)上)标注 注解

```java
@Component
@Transactional(rollbackFor = Exception.class)
public class PayServiceImpl implements PayService {

    @Autowired
    private AccountInfoDao accountInfoDao;

    @Autowired
    private ProductInfoDao productInfoDao;
    
    public void pay(String accountId, double money) {
        //查询余额
        double blance = accountInfoDao.qryBlanceByUserId(accountId);

        //余额不足正常逻辑
        if(new BigDecimal(blance).compareTo(new BigDecimal(money))<0) {
            throw new RuntimeException("余额不足");
        }

        //更新库存
//        ((PayService) AopContext.currentProxy()).updateProductStore(1);
        productInfoDao.updateProductInfo(1);


        System.out.println(1/0);

        //更新余额
        int retVal = accountInfoDao.updateAccountBlance(accountId,money);
    }

//    @Transactional(propagation =Propagation.REQUIRES_NEW)
    public void updateProductStore(Integer productId) {
        try{
            productInfoDao.updateProductInfo(productId);
        }
        catch (Exception e) {
            throw new RuntimeException("内部异常");
        }
    }
}
```

##### 3.运行测试

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MainConfig.class);

    PayService payService = context.getBean(PayService.class);
    payService.pay("123456789",10);

}
```

就这么简单，事务就生效了(这条数据并没有insert成功，测试时特意放了1/0的函数)。

#### 原理

接下来分析注解驱动事务的原理，同样的我们从@EnableTransactionManagement开始:

（之后会补流程图）

##### @EnableTransactionManagement

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
```

简直不要太面熟好不好，属性和**@EnableAspectJAutoProxy**注解一个套路。不同之处只在于**@Import**导入器导入 的这个类，不同的是:它导入的是个ImportSelector

##### TransactionManagementConfigurationSelector

它所在的包为org.springframework.transaction.annotation，jar属于:**spring-tx**(若引入了spring-jdbc，这个jar 会自动导入)

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

   /**
    * Returns {@link ProxyTransactionManagementConfiguration} or
    * {@code AspectJ(Jta)TransactionManagementConfiguration} for {@code PROXY}
    * and {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()},
    * respectively.
    */
   @Override
   protected String[] selectImports(AdviceMode adviceMode) {
      switch (adviceMode) {
          // 很显然，绝大部分情况下，我们都不会使用AspectJ的静态代理的~~~~~~~~
					// 这里面会导入两个类~~~
         case PROXY:
            return new String[] {AutoProxyRegistrar.class.getName(),
                  ProxyTransactionManagementConfiguration.class.getName()};
         case ASPECTJ:
            return new String[] {determineTransactionAspectClass()};
         default:
            return null;
      }
   }

   private String determineTransactionAspectClass() {
      return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
            TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
            TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
   }

}
```

AdviceModeImportSelector目前所知的三个子类是:

AsyncConfigurationSelector、TransactionManagementConfigurationSelector、CachingConfigurationSelector。

由此可见后面还会着重分析的Spring的缓存体系@EnableCaching，和异步@EnableAsync模式也是和这个极其类似的。

##### AutoProxyRegistrar

从名字是意思是:自动代理注册器。它是个ImportBeanDefinitionRegistrar，可以实现自己向容器里注册Bean的 定义信息

```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   private final Log logger = LogFactory.getLog(getClass());

   /**
    * Register, escalate, and configure the standard auto proxy creator (APC) against the
    * given registry. Works by finding the nearest annotation declared on the importing
    * {@code @Configuration} class that has both {@code mode} and {@code proxyTargetClass}
    * attributes. If {@code mode} is set to {@code PROXY}, the APC is registered; if
    * {@code proxyTargetClass} is set to {@code true}, then the APC is forced to use
    * subclass (CGLIB) proxying.
    * <p>Several {@code @Enable*} annotations expose both {@code mode} and
    * {@code proxyTargetClass} attributes. It is important to note that most of these
    * capabilities end up sharing a {@linkplain AopConfigUtils#AUTO_PROXY_CREATOR_BEAN_NAME
    * single APC}. For this reason, this implementation doesn't "care" exactly which
    * annotation it finds -- as long as it exposes the right {@code mode} and
    * {@code proxyTargetClass} attributes, the APC can be registered and configured all
    * the same.
    */
   @Override
   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      boolean candidateFound = false;
     // 这里面需要特别注意的是:这里是拿到所有的注解类型~~~而不是只拿@EnableAspectJAutoProxy这个类型的 
     // 原因:因为mode、proxyTargetClass等属性会直接影响到代理得方式，而拥有这些属性的注解至少有:
	// @EnableTransactionManagement、@EnableAsync、@EnableCaching等~~~~
	// 甚至还有启用AOP的注解:@EnableAspectJAutoProxy它也能设置`proxyTargetClass`这个属性的值，因此也会产生关联影响~
      Set<String> annTypes = importingClassMetadata.getAnnotationTypes();
      for (String annType : annTypes) {
         AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
         if (candidate == null) {
            continue;
         }
        // 拿到注解里的这两个属性
				// 说明:如果你是比如@Configuration或者别的注解的话 他们就是null了
         Object mode = candidate.get("mode");
         Object proxyTargetClass = candidate.get("proxyTargetClass");
        // 如果存在mode且存在proxyTargetClass 属性
				// 并且两个属性的class类型也是对的，才会进来此处(因此其余注解相当于都挡外面了~)
         if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
               Boolean.class == proxyTargetClass.getClass()) {
           // 标志:找到了候选的注解~~~~
            candidateFound = true;
            if (mode == AdviceMode.PROXY) {
              // 这一部是非常重要的~~~~又到了我们熟悉的AopConfigUtils工具类，且是熟悉的registerAutoProxyCreatorIfNecesary方法
              // 它主要是注册了一个`internalAutoProxyCreator`，但是若出现多次的话，这里不是覆盖的形式，而是以第一次的为主
							// 当然它内部有做等级的提升之类的，这个之前也有分析过~~~~
               AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
               if ((Boolean) proxyTargetClass) {
                 // 看要不要强制使用CGLIB的方式(由此可以发现 这个属性若出现多次，是会是覆盖的形式)
                  AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                  return;
               }
            }
         }
      }
      if (!candidateFound && logger.isInfoEnabled()) {
         String name = getClass().getSimpleName();
         logger.info(String.format("%s was imported but no annotations were found " +
               "having both 'mode' and 'proxyTargetClass' attributes of type " +
               "AdviceMode and boolean respectively. This means that auto proxy " +
               "creator registration and configuration may not have occurred as " +
               "intended, and components may not be proxied as expected. Check to " +
               "ensure that %s has been @Import'ed on the same class where these " +
               "annotations are declared; otherwise remove the import of %s " +
               "altogether.", name, name, name));
      }
   }

}
```

这一步最重要的就是向Spring容器注入了一个自动代理创建器: org.springframework.aop.config.internalAutoProxyCreator，这里有个小细节注意一下，由于AOP和事务注册的都是名字为org.springframework.aop.config.internalAutoProxyCreator 的BeanPostProcessor，但是只会保留一个，AOP的会 覆盖事务的， 因为AOP优先级更大

```java
@Nullable
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      return null;
   }

   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```

所以假如@EnableTransactionManagement和@EnableAspectJAutoProxy 同时存在， 那么AOP的AutoProxyCreator会进行覆盖。

##### ProxyTransactionManagementConfiguration

它是一个@Configuration,所以看看它向容器里注入了哪些Bean

```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

  // 这个Advisor可是事务的核心内容
   @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
         TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {

      BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
      advisor.setTransactionAttributeSource(transactionAttributeSource);
      advisor.setAdvice(transactionInterceptor);
      if (this.enableTx != null) {
        // 顺序由@EnableTransactionManagement注解的Order属性来指定 默认值为:Ordered.LOWEST_PRECEDENCE
         advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
      }
      return advisor;
   }

  // TransactionAttributeSource 这种类特别像 `TargetSource`这种类的设计模式
	// 这里直接使用的是AnnotationTransactionAttributeSource 基于注解的事务属性源~~~
   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionAttributeSource transactionAttributeSource() {
      return new AnnotationTransactionAttributeSource();
   }

  // 事务拦截器，它是个`MethodInterceptor`，它也是Spring处理事务最为核心的部分
// 请注意:你可以自己定义一个TransactionInterceptor(同名的)，来覆盖此Bean(注意是覆盖)
// 另外请注意:你自定义的BeanName必须同名，也就是必须名为:transactionInterceptor 否则两个都会注册进容器里 面去~~~~~~
   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {
      TransactionInterceptor interceptor = new TransactionInterceptor();
     // 事务的属性
      interceptor.setTransactionAttributeSource(transactionAttributeSource);
     // 事务管理器(也就是注解最终需要使用的事务管理器,父类已经处理好了)
			// 此处注意:我们是可议不用特殊指定的，最终它自己会去容器匹配一个适合的~~~~
      if (this.txManager != null) {
         interceptor.setTransactionManager(this.txManager);
      }
      return interceptor;
   }

}
```

父类(抽象类) 它实现了ImportAware接口 所以拿到@Import所在类的所有注解信息

```java
@Configuration
public abstract class AbstractTransactionManagementConfiguration implements ImportAware {

   @Nullable
   protected AnnotationAttributes enableTx;

   /**
    * Default transaction manager, as configured through a {@link TransactionManagementConfigurer}.
    */
  	// 此处:注解的默认的事务处理器(可议通过实现接口TransactionManagementConfigurer来自定义配置)
		// 因为事务管理器这个东西，一般来说全局一个就行，但是Spring也提供了定制化的能力~~~
   @Nullable
   protected TransactionManager txManager;


   @Override
   public void setImportMetadata(AnnotationMetadata importMetadata) {
     // 此处:只拿到@EnableTransactionManagement这个注解的就成~~~~~ 作为AnnotationAttributes保存起来
      this.enableTx = AnnotationAttributes.fromMap(
            importMetadata.getAnnotationAttributes(EnableTransactionManagement.class.getName(), false));
      // 这个注解是必须的~~~~~~~~~~~~~~~~
     if (this.enableTx == null) {
         throw new IllegalArgumentException(
               "@EnableTransactionManagement is not present on importing class " + importMetadata.getClassName());
      }
   }

  // 可以配置一个Bean实现这个接口。然后给注解驱动的给一个默认的事务管理器~~~~
	// 设计模式都是想通的~~~
   @Autowired(required = false)
   void setConfigurers(Collection<TransactionManagementConfigurer> configurers) {
      if (CollectionUtils.isEmpty(configurers)) {
         return;
      }
     // 同样的，最多也只允许你去配置一个~~~
      if (configurers.size() > 1) {
         throw new IllegalStateException("Only one TransactionManagementConfigurer may exist");
      }
      TransactionManagementConfigurer configurer = configurers.iterator().next();
      this.txManager = configurer.annotationDrivenTransactionManager();
   }


  // 注册一个监听器工厂，用以支持@TransactionalEventListener注解标注的方法，来监听事务相关的事件
 // 通过事件监听模式来实现事务的监控~~~~
   @Bean(name = TransactionManagementConfigUtils.TRANSACTIONAL_EVENT_LISTENER_FACTORY_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public static TransactionalEventListenerFactory transactionalEventListenerFactory() {
      return new TransactionalEventListenerFactory();
   }

}
```

下面最主要的就是分析**BeanFactoryTransactionAttributeSourceAdvisor**这个增强器

##### BeanFactoryTransactionAttributeSourceAdvisor

首先看它的父类: 它是一个和Bean工厂和事务都有关系的Advisor 从上面的配置类可议看出，它是new出来一个。 使用的Advice为:advisor.setAdvice(transactionInterceptor())，即容器内的事务拦截器~~~~

使用的事务属性源为:advisor.setTransactionAttributeSource(transactionAttributeSource())，即一个new      AnnotationTransactionAttributeSource()

支持三种事务注解来标注~~~

```java
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

   @Nullable
   private TransactionAttributeSource transactionAttributeSource;

  // 这个很重要，就是切面。它决定了哪些类会被切入，从而生成的代理对象~
 // 关于:TransactionAttributeSourcePointcut 下面有说~
   private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
      // 注意此处`getTransactionAttributeSource`就是它的一个抽象方法~~~
     @Override
      @Nullable
      protected TransactionAttributeSource getTransactionAttributeSource() {
         return transactionAttributeSource;
      }
   };


   /**
    * Set the transaction attribute source which is used to find transaction
    * attributes. This should usually be identical to the source reference
    * set on the transaction interceptor itself.
    * @see TransactionInterceptor#setTransactionAttributeSource
    */
   // 可议手动设置一个事务属性源~
   public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
      this.transactionAttributeSource = transactionAttributeSource;
   }

   /**
    * Set the {@link ClassFilter} to use for this pointcut.
    * Default is {@link ClassFilter#TRUE}.
    */
  // 当然我们可以指定ClassFilter 默认情况下:ClassFilter classFilter = ClassFilter.TRUE; 匹配所有的类的
   public void setClassFilter(ClassFilter classFilter) {
      this.pointcut.setClassFilter(classFilter);
   }

  // 此处pointcut就是使用自己的这个pointcut去切入~~~
   @Override
   public Pointcut getPointcut() {
      return this.pointcut;
   }

}
```

下面当然要重点看看TransactionAttributeSourcePointcut，它是怎么切入的 TransactionAttributeSourcePointcut 这个就是事务的匹配Pointcut切面，决定了哪些类需要生成代理对象从而应用事务。

```java
// 首先它的访问权限事default 显示是给内部使用的
// 首先它继承自StaticMethodMatcherPointcut 所以`ClassFilter classFilter = ClassFilter.TRUE;` 匹配所有的类
// 并且isRuntime=false 表示只需要对方法进行静态匹配即可~~~~
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

   protected TransactionAttributeSourcePointcut() {
      setClassFilter(new TransactionAttributeSourceClassFilter());
   }

  // 方法的匹配 静态匹配即可(因为事务无需要动态匹配这么细粒度~~~)
   @Override
   public boolean matches(Method method, Class<?> targetClass) {
     // 重要:拿到事务属性源~~~~~~
 // 如果tas == null表示没有配置事务属性源，那是全部匹配的 也就是说所有的方法都匹配~~~~(这个处理还是比较让我诧异的~~~)
 // 或者 标注了@Transaction这样的注解的方法才会给与匹配~~~
      TransactionAttributeSource tas = getTransactionAttributeSource();
      return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
   }

   @Override
   public boolean equals(@Nullable Object other) {
      if (this == other) {
         return true;
      }
      if (!(other instanceof TransactionAttributeSourcePointcut)) {
         return false;
      }
      TransactionAttributeSourcePointcut otherPc = (TransactionAttributeSourcePointcut) other;
      return ObjectUtils.nullSafeEquals(getTransactionAttributeSource(), otherPc.getTransactionAttributeSource());
   }

   @Override
   public int hashCode() {
      return TransactionAttributeSourcePointcut.class.hashCode();
   }

   @Override
   public String toString() {
      return getClass().getName() + ": " + getTransactionAttributeSource();
   }


   /**
    * Obtain the underlying TransactionAttributeSource (may be {@code null}).
    * To be implemented by subclasses.
    */
   @Nullable
   protected abstract TransactionAttributeSource getTransactionAttributeSource();


   /**
    * {@link ClassFilter} that delegates to {@link TransactionAttributeSource#isCandidateClass}
    * for filtering classes whose methods are not worth searching to begin with.
    */
   private class TransactionAttributeSourceClassFilter implements ClassFilter {

     // 实现了如下三个接口的子类，就不需要被代理了 直接放行
      @Override
      public boolean matches(Class<?> clazz) {
         if (TransactionalProxy.class.isAssignableFrom(clazz) ||
               TransactionManager.class.isAssignableFrom(clazz) ||
               PersistenceExceptionTranslator.class.isAssignableFrom(clazz)) {
            return false;
         }
         TransactionAttributeSource tas = getTransactionAttributeSource();
         return (tas == null || tas.isCandidateClass(clazz));
      }
   }

}
```

关于matches方法的调用时机：只要容器内的每个bean，都会经过AbstractAutoProxyCreator#postProessAfterInitialization从而会调用wrapIfNecessary方法，因此容器内所有的bean的所有方法在容器启动的时候都会执行matches方法，因此注意缓存的使用。

##### 解析advisor

在Spring AOP中有过过介绍，解析事务advisor详细代码:

org.springframework.aop.framework.autoproxy.BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans

```java
/**
 * Find all eligible Advisor beans in the current bean factory,
 * ignoring FactoryBeans and excluding beans that are currently in creation.
 * @return the list of {@link org.springframework.aop.Advisor} beans
 * @see #isEligibleBean
 */
public List<Advisor> findAdvisorBeans() {
   // Determine list of advisor bean names, if not cached already.
/**
* 探测器字段缓存中cachedAdvisorBeanNames 是用来保存我们的Advisor全类名 
* 会在第一个单实例bean的中会去把这个advisor名称解析出来
*/
   String[] advisorNames = this.cachedAdvisorBeanNames;
   if (advisorNames == null) {
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the auto-proxy creator apply to them!
 /**
* 去我们的容器中获取到实现了Advisor接口的实现类
* 而我们的事务注解@EnableTransactionManagement 导入了一个叫ProxyTransactionManagementConfiguration配置
* 而在这个配置类中配置了
* @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME) * *@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
*public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(); 然后把他的名字获取出来保存到 本类的属性变量*cachedAdvisorBeanNames中
*/
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      this.cachedAdvisorBeanNames = advisorNames;
   }
  //若在容器中没有找到，直接返回一个空的集合
   if (advisorNames.length == 0) {
      return new ArrayList<>();
   }

   List<Advisor> advisors = new ArrayList<>();
  //ioc容器中找到了我们配置的BeanFactoryTransactionAttributeSourceAdvisor
   for (String name : advisorNames) {
      //判断他是不是一个合适的 是我们想要的 默认true
      if (isEligibleBean(name)) {
         //BeanFactoryTransactionAttributeSourceAdvisor是不是正在创建的bean
         if (this.beanFactory.isCurrentlyInCreation(name)) {
            if (logger.isTraceEnabled()) {
               logger.trace("Skipping currently created advisor '" + name + "'");
            }
         }
        //不是的话
         else {
            try {
              //显示的调用getBean方法方法创建我们的BeanFactoryTransactionAttributeSourceAdvisor返回去
               advisors.add(this.beanFactory.getBean(name, Advisor.class));
            }
            catch (BeanCreationException ex) {
               Throwable rootCause = ex.getMostSpecificCause();
               if (rootCause instanceof BeanCurrentlyInCreationException) {
                  BeanCreationException bce = (BeanCreationException) rootCause;
                  String bceBeanName = bce.getBeanName();
                  if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                     if (logger.isTraceEnabled()) {
                        logger.trace("Skipping advisor '" + name +
                              "' with dependency on currently created bean: " + ex.getMessage());
                     }
                     // Ignore: indicates a reference back to the bean we're trying to advise.
                     // We want to find advisors other than the currently created bean itself.
                     continue;
                  }
               }
               throw ex;
            }
         }
      }
   }
   return advisors;
}
```

##### 创建动态代理

在Spring AOP中有过过介绍，区别在于匹配方式的不同: 

- AOP是按照Aspectj提供的API 结合切点表达式进行匹配。 
- 事务是根据方法或者类或者接口上面的@Transactional进行匹配。

所以本文和aop重复的就省略了如下:

![截屏2020-10-16 16.01.42](./1.1.png)

```java
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
   Assert.notNull(pc, "Pointcut must not be null");
  // 进行类级别过滤 这里会返回
   if (!pc.getClassFilter().matches(targetClass)) {
      return false;
   }

   /**
 	* 进行方法级别过滤
 	*/
  //如果pc.getMethodMatcher()返回TrueMethodMatcher则匹配所有方法
   MethodMatcher methodMatcher = pc.getMethodMatcher();
  
   if (methodMatcher == MethodMatcher.TRUE) {
      // No need to iterate the methods if we're matching any method anyway...
      return true;
   }

   IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
  //判断匹配器是不是IntroductionAwareMethodMatcher 只有AspectJExpressionPointCut才会实现这个接口
   if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
      introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
   }

  //创建一个集合用于保存targetClass 的class对象
   Set<Class<?>> classes = new LinkedHashSet<>();
   if (!Proxy.isProxyClass(targetClass)) {
     //加入到集合中去
      classes.add(ClassUtils.getUserClass(targetClass));
   }
  //获取到targetClass所实现的接口的class对象，然后加入到集合中
   classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
	//循环所有的class对象
   for (Class<?> clazz : classes) {
     //通过class获取到所有的方法
      Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
      for (Method method : methods) {
         //通过methodMatcher.matches来匹配我们的方法
         if (introductionAwareMethodMatcher != null ?
             // 通过切点表达式进行匹配 AspectJ方式
               introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
              // 通过方法匹配器进行匹配 内置aop接口方式
               methodMatcher.matches(method, targetClass)) {
            // 只要有1个方法匹配上了就创建代理
            return true;
         }
      }
   }

   return false;
}
```

关键点:

- if (!pc.getClassFilter().matches(targetClass)) {
   初筛时事务不像aop， 上面介绍的TransactionAttributeSourcePointcut的getClassFilter是TrueClassFilter。所以所有的类都能匹配

- if (methodMatcher instanceof IntroductionAwareMethodMatcher) {

  事务的methodMatcher 没有实现该接口。 只有AOP的实现了该接口所以也导致下面: 

- methodMatcher.matches(method, targetClass)

  所以事务时直接调用methodMatcher.matches进行匹配

##### 匹配方式

###### 1.org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut#matches

```java
public boolean matches(Class<?> clazz) {
   if (TransactionalProxy.class.isAssignableFrom(clazz) ||
         TransactionManager.class.isAssignableFrom(clazz) ||
         PersistenceExceptionTranslator.class.isAssignableFrom(clazz)) {
      return false;
   }
  
/**
* 获取我们@EnableTransactionManagement注解为我们容器中导入的ProxyTransactionManagementConfiguration 
* 配置类中的TransactionAttributeSource对象
*/
   TransactionAttributeSource tas = getTransactionAttributeSource();
  // 通过getTransactionAttribute看是否有@Transactional注解
   return (tas == null || tas.isCandidateClass(clazz));
}
```

关键点

- TransactionAttributeSource tas = getTransactionAttributeSource();
  - 这里获取到的时候 通过@Import 的ImportSelect 注册的配置类 ProxyTransactionManagementConfiguration 中设置的 AnnotationTransactionAttributeSource:它是基于注解驱动的事务管理的事务属性源， 和@Transaction 相关，也是现在使用得最最多的方式。

它的基本作用为:它遇上比如 @Transaction标注的方法时，此类会分析此事务注解，最终组织形成一个TransactionAttribute供随后的调用。

- NameMatchTransactionAttributeSource:根据名字就能匹配，然后该事务属性 就会作用在对应的方法上。
- MethodMapTransactionAttributeSource:它的使用方式和 NameMatchTransactionAttributeSource基本相同,指定具体方法名
- CompositeTransactionAttributeSource :组合模式

###### 2.org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource#getTransactionAttribute

```java
@Override
@Nullable
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
  //判断method所在的class 是不是Object类型
  if (method.getDeclaringClass() == Object.class) {
      return null;
   }

   // First, see if we have a cached value.
  //构建我们的缓存key
   Object cacheKey = getCacheKey(method, targetClass);
  //先去我们的缓存中获取
   TransactionAttribute cached = this.attributeCache.get(cacheKey);
  //缓存中不为空
   if (cached != null) {
      // Value will either be canonical value indicating there is no transaction attribute,
      // or an actual transaction attribute.
     //判断缓存中的对象是不是空事务属性的对象
      if (cached == NULL_TRANSACTION_ATTRIBUTE) {
         return null;
      }
     //不是的话 就进行返回
      else {
         return cached;
      }
   }
   else {
      // We need to work it out.
      //我们需要查找我们的事务注解 匹配在这体现
      TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
      // Put it in the cache.
     // 若解析出来的事务注解属性为空
      if (txAttr == null) {
        //往缓存中存放空事务注解属性
         this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
      }
      else {
        //我们执行方法的描述符 全类名+方法名
         String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
        //把方法描述设置到事务属性上去
         if (txAttr instanceof DefaultTransactionAttribute) {
            ((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
         }
         if (logger.isTraceEnabled()) {
            logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
         }
        //加入到缓存
         this.attributeCache.put(cacheKey, txAttr);
      }
      return txAttr;
   }
}
```

关键点:

- if (cached != null) {
  - 先从缓存中取， 因为这个过程相对比较耗资源，会使用缓存存储已经解析过的， 后续

调用时需要获取

- TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
  - 该方法中具体执行匹配过程 大致是: 实现类方法--->接口的方法--->实现类---->接口类

- ((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
  - 记录当前需要执行事务的方法名，记录到 方便后续调用时判断

- this.attributeCache.put(cacheKey, txAttr);
  - 加入到缓存中

3. ###### 看下是怎么匹配的:

   org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource#computeTransactionAttribute

```java
@Nullable
protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
   // Don't allow no-public methods as required.
   //判断我们的事务方法上的修饰符是不是public的
   if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
      return null;
   }

   // The method may be on an interface, but we need attributes from the target class.
   // If the target class is null, the method will be unchanged.
   Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

   // First try is the method in the target class.
  //第一步，我们先去目标class的方法上去找我们的事务注解
   TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
   if (txAttr != null) {
      return txAttr;
   }

   // Second try is the transaction attribute on the target class.
  //第二步:去我们targetClass类[实现类]上找事务注解
   txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
   if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
      return txAttr;
   }
	// 具体方法不是当前的方法说明 当前方法是接口方法
   if (specificMethod != method) {
      // Fallback is to look at the original method.
      //去我们的实现类的接口上的方法去找事务注解
      txAttr = findTransactionAttribute(method);
      if (txAttr != null) {
         return txAttr;
      }
      // Last fallback is the class of the original method.
     //去我们的实现类的接口上去找事务注解
      txAttr = findTransactionAttribute(method.getDeclaringClass());
      if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
         return txAttr;
      }
   }

   return null;
}
```

关键点:
 这个方法乍一看，觉得是先从实现类方法-->实现类--->接口方法--->接口类 但是!!!! 他在第一 个实现方法查找就已经找了接口方法父类方法。在实现类里面就找了接口类和父类， 所以接口方法--->接

口类 基本走了也没啥用

- Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);
  - 得到具体方法， 如果method是接口方法那将从targetClass得到实现类的方法 ， 所以说无论 传的是接口还是实现， 都会先解析实现类， 所以接口传进来基本没啥用，因为 findTransactionAttribute方法本身就会去接口中解析

- TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
  - 根据具体方法解析

- txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
  - 根据实现类解析

#### @Transactional简单解释

这个事务注解可以用在类上，也可以用在方法上。

- 将事务注解标记到服务组件类级别,相当于为该服务组件的每个服务方法都应用了这个注解 
- 事务注解应用在方法级别，是更细粒度的一种事务注解方式

**另外，Spring支持三种不同的事务注解：**

1. Spring 事务注解 org.springframework.transaction.annotation.Transactional(纯正血统，官方推荐)
2.  JTA事务注解 javax.transaction.Transactional
3. EJB 3 事务注解 javax.ejb.TransactionAttribute 上面三个注解虽然语义上一样，但是使用方式上不完全一样，若真要使其它的时请注意各自的使用方式~

注意 : 如果某个方法和该方法所属类上都有事务注解属性，优先使用方法上的事务注解属性。

