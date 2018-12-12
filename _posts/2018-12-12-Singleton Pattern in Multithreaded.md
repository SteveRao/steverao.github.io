---
layout:     post
title:      "多线程环境中的单例模式"
subtitle:   "单例模式，线程安全"
date:       2018-12-12
author:     "ZihaoRao"
header-img: "img/post-bg-sparksql.jpg"

tags: 设计模式，单例模式，多线程
---





#### 总体概述

> 单例模式作为设计模式中较为简单但最常用的一种。理解其思想相对容易。但如何在开发中合理地将其应用却是需要勤加练习的！特别是对有状态属性的单例类如何确保其在多线程环境中"线程安全"地为系统提供服务是非常值得思考的问题。   ​                                                                                                                                            
>



#### 单例模式速览

------

1. ##### 定义

   单例模式(`Singleton Pattern`)作为`java`中的一种**创建型**设计模式，为对象的创建提供了一种非常有价值的借鉴方式。单例类实例作为整个系统中的该类唯一实例，它具有服务系统全局的能力。

2. ##### 关键点

   （1）单例类只有唯一一个实例，所以要将其构造函数声明成`private`，禁止外部直接创建。

   （2）单例类必须自己创建自己的唯一实例，所以对象的初始化在类中进行。

   （3）单例类必须给所有其他对象提供该实例对象。所以类中要有获取对象实例的静态方法。



#### 四种单例模式实现方式

------

1. ##### 懒汉模式

   ```jade
   public class Singleton {  
       /*该处不需要volatile关键字，因为synchronized已可实现变量共享 */
       private static Singleton singleton; 
       private Singleton (){}  
       public static Singleton getSingleton() {  
       if (singleton == null) {  
           synchronized (Singleton.class) {  
           if (singleton == null) {  
               /*初始化代码，第一次使用时才初始化*/
               singleton = new Singleton();  
           }  
           }  
       }  
       return singleton;  
       }  
   }
   ```

2. **饿汉模式**

   ```jade
   public class Singleton {  
       /*初始化代码，类加载时就初始化静态属性*/
       private static Singleton instance = new Singleton();  
       private Singleton (){}  
       public static Singleton getInstance() {  
           return instance;  
       }  
   }
   ```

   从上述实例化的时刻便可理解这两个名字的由来，**“饿汉”**顾名思义一开始就很饥饿所以很着急地在初始化阶段就创建该单例类的唯一实例。而**“懒汉”**则表示直到有第一次实例的调用请求才初始化该唯一单实例，所以其对初始化行为非常的懒散。**因此单例模式在初始化实例时所面临的“非线程安全”问题也只有在“懒汉”模式中才会发生！**

3. **静态内部类**

   ```jade
   public class Singleton {  
       /*静态内部类延迟初始化*/
       private static class SingletonHolder {  
           private static final Singleton INSTANCE = new Singleton();  
       }  
       private Singleton (){}  
       public static final Singleton getInstance() {  
           return SingletonHolder.INSTANCE;  
       }  
   }
   ```

   静态内部类结合了“饿汉“和“懒汉”模式各自的有点，摒弃他们的不足，即达到了**延迟加载避免资源的过早或不必要消耗**，又通过内部类初始化避免了”线程安全“问题。

4. **枚举**

   ```jade
   public enum Singleton {  
       INSTANCE;  
       public void whateverMethod() {  
       }  
   }
   ```

   这种使用枚举来实现单例模式的方式最简洁，其效果也很好。不需要私有化构造函数，枚举类构造函数本身就私有化，还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化的发生。因此它是一种推荐使用的方式。



**单例模式与线程安全**

------

1. **概念解析**

   **线程安全：**“非线程安全”指多个线程对同一对象中的同一个实例进行操作时会出现值被修改、值不同步等情况所引起的程序运行冲突。相反，则表明线程安全。

   **多线程中使用单例模式的条件：**一个类能够以单实例的方式运行的前提是**“无状态”**，即一个类不能拥有有状态的属性。否则，在系统不同处对其无序调用修改状态变量后就可能会出现“非线程安全”的情况。

2. **问题背景**

   一般简单的单例类可能不会有“状态”属性，但在稍微复杂一点的系统中单例类拥有“状态”属性是很常见的。例如，通过构造一个单例类实现数据库连接池功能。其中，因为`DAO`可能会拥有不同的`Connection`连接状态，所以，该连接池要管理不同状态的连接。连接对该连接池来说是一个状态变量，如何在多线程环境中仍然保证该拥有状态属性的连接池在单例的形式下保证线程安全就非常关键。

3. **解决策略**

   采用“去状态化”思想，通过将单例类中不同状态与对应的线程关联起来，实现不同线程访问该单例对象时仅调用其对应的状态， 这种将状态本地线程化的方法即可达到将单例类中的状态变量“去状态化”。`Java`中提供的`ThreadLocal`类就能很好的达到该目的。例如：`Spring`中就是使用`ThreadLocal`来保证`DAO`和`Service`类都可以以单实例的方式存在。

4. **缓存案例剖析**

   最近做的`BI可视化系统`就有这样一个需求，用户可以建立多个同种数据源不同数据库的对应数据集。当用户如果反复切换不同数据集进行数据查看，就可能总是需要销毁上一个数据库连接池新建下一个。这是非常消耗资源的操作！所以我打算通过创建一个单例类作为缓存，将创建了的连接池进行缓存避免不必要的开销！

   ```jade
   /*针对Cassandra数据库连接池创建的缓存，这里直接使用枚举方式创建单例类*/
   public enum ConnectionProvider implements ConnectionManager<Session, CassandraDataSource> {
   
       INSTANCE;
       /*连接池缓存容器*/
       public static final ConcurrentHashMap<CassandraConnectInfo, Cluster> CACHE = new ConcurrentHashMap(10);
    
       /*获取连接方法*/
       private Session getConnection(CassandraConnectInfo connectInfo) throws DataSourceException {
           /*每一次获取连接之前都先根据连接信息看容器中是否有缓存，有则取，无则新建并缓存*/
           Cluster cluster = Optional.ofNullable(CACHE.get(connectInfo)).orElseGet(() -> instanceSession(connectInfo));
           return cluster.connect();
       }
       /*其他代码省略*/
   }
   ```

   因为该缓存仅需要一个并能为整个系统提供服务，所以必须是单例的！然后通过类似于`ThreadLocal`的思想比较连接信息，就可实现连接的缓存和获取，通过设计一个这样的缓存单例，就能很好的避免陷入反复创建和销毁连接的囧境！



#### 小结

------

单例类确实能避免常用的**复杂对象**的反复初始化和销毁从而导致的资源过度消耗！但如果仅仅认为其的作用就在于此那就太局限了！其最佳的使用场景应该是在那种只有使用单例模式才能解决问题的场合下。例如在单例类中配置整个项目有关的全局信息，系统全局唯一的数据库连接池和上例所示连接池缓存等。这些问题只有使用单例才能解决！



#### 参考文献

------

1. [菜鸟教程之单例模式](http://www.runoob.com/design-pattern/singleton-pattern.html)
2. 《精通Spring 4.X企业应用开发实践》第十一章中的ThreadLocal基础知识
3. 《Java多线程编程核心技术》第六章单例模式与多线程


