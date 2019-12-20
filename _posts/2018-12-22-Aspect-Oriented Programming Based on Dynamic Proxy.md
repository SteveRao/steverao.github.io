---
layout:     post
title:      "由动态代理到面向切面编程(AOP)"
subtitle:   "面向切面编程(AOP)之动态代理"
date:       2018-12-22
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 设计模式
---





### 总体概述
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2018/12/22/Aspect-Oriented-Programming-Based-on-Dynamic-Proxy/)
>
> 相比于**继承**这种`Java`自提供的*纵向*抽取子类相同部分构造父类，达到代码复用目的的方式。`代理`或者`AOP`为*横向*的代码冗余提供了一种解决方法。其终极目的是为了实现代码的复用与解耦。                                                                                                                                               



### 1.代理总述
---
1. ##### 代理设计模式

   `代理设计模式`作为`Java`提供的一种结构型模式，代理类帮助所代理的对象完成一些与其核心功能无关的繁杂事务，还被代理对象一个逻辑清晰的世界。

2. ##### 代理意义

   相比于**继承**这种`Java`自提供的纵向抽取子类相同部分构造父类，达到代码复用目的的方式。代理或者`AOP`为横向的代码冗余提供了一种解决方法。其终极目的是为了实现代码的复用与解耦。详情见下图：

   ![proxy-logic picture](/img/in-post/content/proxy-logic.png)

   左图在不使用代理时，每个要进行监控的类，都需要在业务逻辑的前后通过硬编码的方式实现监控功能，而在采用了`动态代理（AOP）`后便可实现将监控代码从每个业务类中抽取出来封装成一个整体，让各个需要监控服务的业务类仅关注自身业务逻辑的实现不考虑监控这一统一非具体业务功能。最后在功能调用时再`动态织入`监控代码。这样就很好的实现了代码复用与解耦。以上便是我对`动态代理`以及其衍生技术`AOP`的一个大致理解，详细的相关实现案例见下文。在正式介绍动态代理之前还是先简单说明一下静态代理。


### 2.代理实现——静态代理
---

1. **基本概念：**为需要代理的**某一个类**写一个代理行为。
2. **案例说明：**[静态代理模式](http://www.runoob.com/design-pattern/proxy-pattern.html)
3. **扩展理解：**类比日常生活中房屋中介为房主进行租赁代理，律师为客户进行法律事务代理等例子。可发现只要目标对象在特定的情况下不便对外直接提供服务，都可以为其创建一个代理类来帮助其更好的完成对外交互。
4. **疑难解析：**代理类其实就只是替被代理类代理了若干的方法，所有它需要继承被代理类的父类，以便让其拥有与被代理类一样的对外接口。代理类中需被代理的方法一定是继承或者重写的父类方法。



### 3.代理实现——动态代理
---

1. **基本概念：**为需要代理的**一组类**写一个代理行为。
2. **Java中支持的动态代理模式：**`JDK动态代理` 和 `CGLib动态代理`

3. **JDK动态代理：**`Java1.3`开始提供该支持，主要有`Proxy`和`InvocationHandler`（接口）两个类。以下例子是为一些需要在方法执行前后进行性能监控的类创建的动态代理类案例：

   ```java
   /*被代理类接口*/
   public interface Waiter {
       void greetTo(String clientName);
   }

   /*被代理类*/
   public class NaiveWaiter implements Waiter {
       public void greetTo(String clientName) {
           System.out.println("NaiveWaiter:greet to " + clientName + "...");
       }

       /*父类中没有，子类中另外添加的方法，该方法不能被代理*/
       public void smile(String clientName, int times) {
           System.out.println("NaiveWaiter:smile to  " 
                              + clientName + times 
                              + "times...");
       }
   }

   public class PerformanceHandler implements InvocationHandler {

       private Object target;
       public PerformanceHandler(Object target) {
           this.target = target;
       }

       @Override
       public Object invoke(Object proxy, Method method, Object[] args) 
           throws Throwable {
           /*代理类为业务类提供的特殊代理服务*/
           System.out.println("Monitor begin!!!");
           /*通过反射调用业务类的目标方法*/
           Object obj = method.invoke(target, args);
           /*代理类为业务类提供的特殊代理服务*/
           System.out.println("Monitor end!!!");
           return obj;
       }
   }

   public class ForumServiceTest {
      @Test
       public void JDKProxy() {
          /*被代理类目标业务类*/
          NaiveWaiter target = new NaiveWaiter();
          /*将目标业务类与横切代码编织在一起*/
          PerformanceHandler handler = new PerformanceHandler(target);
          /*创建代理对象*/
          Waiter proxy = (Waiter) Proxy.newProxyInstance(
              target.getClass().getClassLoader(), 
              target.getClass().getInterfaces(), 
              handler);
          /*通过代理类为业务类提供方法的被调用*/
          proxy.greetTo("steve");
          /*无法调用，接口中没有申明！由此可得出JKD动态代理的局限性*/
          //proxy.show()
      }
   }
   ```

   运行结果输出：
   ```
   Monitor begin!!!
   NaiveWaiter:greet to steve...
   Monitor end!!!
   ```

&emsp;&emsp;从上述结果可以清楚地看到监控逻辑被成功地织入被代理类业务逻辑方法运行的前后位置。

4. **CGLib动态代理：**通过以上代码及注释应该会发现`JDK动态代理`的局限性，对于没有通过接口定义业务方法的类，`CGLib`为其提供了动态代理的支持

   ```java
   public class CglibProxy implements MethodInterceptor {
       private Enhancer enhancer=new Enhancer();
       public Object getProxy(Class clazz){
           /*设置需要创建子类的父类：被代理类*/
           enhancer.setSuperclass(clazz);
           enhancer.setCallback(this);
           /*通过字节码技术动态创建被代理类子类：代理类*/
           return enhancer.create();
       }

       /*采用方法拦截器拦截所有父类（被代理类）方法的调用并顺势织入横切逻辑*/
       @Override
       public Object intercept(Object o, 
                               Method method, 
                               Object[] args, 
                               MethodProxy methodProxy) throws Throwable {
           /*代理类为业务类提供的特殊代理服务*/
           System.out.println("Monitor begin!!!");
           /*通过代理类调用父类的方法*/
           Object obj = methodProxy.invokeSuper(o, args);
           /*代理类为业务类提供的特殊代理服务*/
           System.out.println("Monitor end!!!");
           return obj;
        }
   }

   public class ForumServiceTest {
   @Test
       public void CGLibProxy() {
           CglibProxy cglibProxy = new CglibProxy();
           /*动态生成子类的方式创建代理类*/
           NaiveWaiter proxy = 
           (NaiveWaiter) cglibProxy.getProxy(NaiveWaiter.class);
           proxy.greetTo("steve");
           /*由于代理类是目标类的子类所有可以调用目标类相对于其父类中额外添加的方法*/
           proxy.smile("alan", 10);
        }
   }
   ```

   运行结果输出：

   ```java
   Monitor begin!!!
   NaiveWaiter:greet to steve...
   Monitor end!!!
   Monitor begin!!!
   NaiveWaiter:smile to  alan 10 times...
   Monitor end!!!
   ```

   从上述例子可知，只要是属于被代理类中的方法( greetTo() 和 smile() )，都可以被成功的织入监视逻辑代码。

5. **JDK动态代理与CGLib动态代理小结：**

   |    代理方式     | 代理对象  |            性能            | 代理与被代理间关系 |
   | :---------: | :---: | :----------------------: | :-------: |
   |  `JDK动态代理`  | 需实现接口 |    创建过程简单，适合需要频繁创建的场景    | **兄弟类关系** |
   | `CGLib动态代理` |  无要求  | 创建过程复杂但代理类性能好，适合为单例类创建代理 | **继承关系**  |





### 4.基于动态代理的面向切面编程（AOP Aspect Oriented Programing）
---

1. **面向切面编程引入：**相比于继承实现的纵向代码复用，动态代理解决了一个横向的代码复用难题，但动态代理在织入横切逻辑方面不够灵活。正因此，`面向切面编程(AOP)`在动态代理技术基础上，通过增加连接点、切点和增强等概念让织入横切逻辑变更加灵活与可控！

2. **核心术语：**

   （1）***连接点：***一个类或一段代码拥有一些具有边界性质的特定点，这些代码中的特定点被称作“连接点”。`Spring`仅支持方法上的连接点，即只能在方法调用前、调用后、抛出异常时及方法调用前后这些程序执行点织入增强。

   （2）***切点：***筛选特定连接点的规则，连接点与切点的关系就相当于数据库中的数据和查询条件。

   （3）***增强：***织入目标类连接点上的一段程序代码，类似于上例中的性能监控Monitor。

   （4）***切面：***其由`切点`和`增强`组成，它既包括横切逻辑的定义，也包括连接点的定义。

3. **黄金等式：**

   ![equation](/img/in-post/content/proxy-equation.png)


   公式一中的`切点`：就表示一组规则，规定增强需要织入哪些特定的类及其特定方法，公式二中的`连接点信息`表示的是在特定方法的具体位置，例如：方法调用前或后等。如果能将上述公式关系弄清楚，就算基本上理解了面向切面编程的逻辑。

4. **案例分析**

   （1）上述的`JDK`和`CGLib动态代理`其实就是两个最简单的面向切面编程案例，由：**增强=横切代码+连接点信息**，可知其中的横切代码就是方法执行前后的监控程序，连接点信息就是被代理类的代理方法的执行前后，所以上述动态代理中`PerformanceHandler .invoke()`和`CglibProxy.interceptor()`方法就属于两个动态代理增强，又由：**切面=切点+增强可知**，两个动态代理中的切点描述的是被代理类的所有方法，因此这两个动态代理增强也是两个简单切面。

   （2）切面分成三种：**一般切面**、**切点切面**和**引介切面**，一般切面就是（1）中提到的只有增强的切面（其切点指的被代理类的所有方法），而切点切面就是添加了详细切点的切面。其中最为特殊的就是引介切面，不像其他切面作用在**类方法级别**上，它是作用在**类级别**上的。它可以为被代理类添加一个接口的实现，因此它能在横向上定义接口的实现方法。

   （3）引介切面实现案例：

   - 被代理类及其将扩展类如下类图

     ![](/img/in-post/content/proxy-class-diagram.png)

   - 如果希望`NaiveWaiter`能同时充当售货员角色，即可通过引介切面技术为`NaiveWaiter`新增`Seller`接口的实现。（以下是采用`@AspectJ`进行`AOP`的案例）

   ```java
   /*使用@Aspect定义一个切面*/
   @Aspect
   public class EnableSellerAspect {
       @DeclareParents(value = "com.raozh.introduction2.NaiveWaiter",
               defaultImpl = SmartSeller.class)
       public Seller seller;
   }

   /*IOCjavaBean配置*/
   <aop:aspectj-autoproxy/>
   <bean id="waiter" class="com.raozh.introduction.NaiveWaiter"/>
   <bean class="com.raozh.introduction.EnableSellerAspect"/>

   /*测试类*/
   public class DeclaredParentsTest {
       public static void main(String[] args) {
            String configPath = "/application.xml";
            ClassPathXmlApplicationContext ctx = 
            new ClassPathXmlApplicationContext(configPath);
            Waiter waiter = (Waiter) ctx.getBean("waiter");
            waiter.serveTo();
           /*将NaiveWaiter转换成Seller*/
            Seller seller = (Seller) waiter;
            seller.sell();
       }
   }
   ```

   运行结果输出:

   ```java
   NaiveWaiter executed serveTo!
   SmartSeller executed sell()!
   ```

   从上述输出结果可知，通过引介切面技术成功为`NaiveWaiter`类扩增了`Seller`接口的实现。引介切面技术为被代理类提供了动态创建新方法和属性的能力，为类的动态扩展提供了技术支持！


### 5.小结
---

- 首先通过介绍代理设计模式讲述代理类的作用与意义，再顺势引入了静态代理和动态代理的概念。
- 重点通过对比介绍`JDK`和`CGLib`两种动态代理的案例展示了动态代理的意义、实现过程和使用要求。
- 最后通过说明动态代理存在的不足，引出面向切面编程技术的作用和特点。
- 另外补充：从上述案例可体会到，`SpringAOP`中提供的面向切面编程的具体操作还是很繁琐的，所以从`JDK1.5`开始`Spring`可以支持`@AspectJ`注解配置的简洁方式实现面向切面编程功能。



### 参考资料
---

- 《精通Spring 4.X企业应用开发实践》第七、八章中的AOP和AspectJ基础知识
- [菜鸟教程之代理模式](http://www.runoob.com/design-pattern/proxy-pattern.html)

