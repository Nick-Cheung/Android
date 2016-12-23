##JAVA 单例模式


###一、前言
　　作为对象的创建模式，单例模式确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例。这个类称为单例类。 

　　单例模式的特点：  
　　1. 单例类只能有一个实例。  
　　2. 单例类必须自己创建自己的唯一实例。   
　　3. 单例类必须给所有其他对象提供这一实例。  
  
  
　　单例模式创建方式分为两大类，饿汉式和懒汉，其中饿汉式比较简单，懒汉式则需要考虑线程安全。

---

###二、饿汉式单例
```java
public class Singleton {
    private static Singleton instance = new Singleton();
    /**
     * 私有默认构造子
     */
    private Singleton(){}
    /**
     * 静态工厂方法
     */
    public static Singleton getInstance(){
        return instance;
    }
}
```
　　上面的例子中，在这个类被加载时，静态变量instance会被初始化，此时类的私有构造子会被调用。这时候，单例类的唯一实例就被创建出来了。

　　饿汉式是典型的空间换时间，当类装载的时候就会创建类的实例，不管你用不用，先创建出来，然后每次调用的时候，就不需要再判断，节省了运行时间。

　　饿汉式单例的缺点是它不是一种懒加载模式（lazy initialization），单例会在加载类后一开始就被初始化，即使客户端没有调用 getInstance()方法。饿汉式的创建方式在一些场景中将无法使用：譬如 Singleton 实例的创建是依赖参数或者配置文件的，在 getInstance() 之前必须调用某个方法设置参数给它，那样这种单例写法就无法使用了。

---

###三、懒汉式单例

　　懒汉式由于是在运行的时候才实例化，因此在多线程运行环境下必须考虑到线程安全的问题，保证单例被正确的创建和获取。


#####1. 懒汉式，线程不安全
```java
public class Singleton {
    private static Singleton instance;
    private Singleton (){}
 
    public static Singleton getInstance() {
        if (instance == null) {
             instance = new Singleton();
        }
        return instance;
    }
}
```
　　这段代码简单明了，而且使用了懒加载模式，但是却存在致命的问题。当有多个线程并行调用 getInstance() 的时候，就会创建多个实例。也就是说在多线程下不能正常工作。

---

#####2. 懒汉式，线程安全

```java
public class Singleton {
    private static Singleton instance;
    private Singleton (){}
 
    public static synchronized Singleton getInstance() {
        if (instance == null) {
           instance = new Singleton();
        }
       return instance;
    } 
}
```
　　为了解决上面的问题，最简单的方法是将整个 getInstance() 方法设为同步（synchronized）。  
  
　　虽然做到了线程安全，并且解决了多实例的问题，但是它并不高效。因为在任何时候只能有一个线程调用 getInstance() 方法。但是同步操作只需要在第一次调用时才被需要，即第一次创建单例实例对象时。这就引出了双重检验锁。  

---
    
#####3. 双重检验锁

```java
public class Singleton {
    private volatile static Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance(){
        //先检查实例是否存在，如果不存在才进入下面的同步块
        if(instance == null){
            //同步块，线程安全的创建实例
            synchronized (Singleton.class) {
                //再次检查实例是否存在，如果不存在才真正的创建实例
                if(instance == null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
　　可以使用“双重检查加锁”的方式来实现，就可以既实现线程安全，又能够使性能不受很大的影响。那么什么是“双重检查加锁”机制呢？

　　所谓“双重检查加锁”机制，指的是：并不是每次进入getInstance方法都需要同步，而是先不同步，进入方法后，先检查实例是否存在，如果不存在才进行下面的同步块，这是第一重检查，进入同步块过后，再次检查实例是否存在，如果不存在，就在同步的情况下创建一个实例，这是第二重检查。这样一来，就只需要同步一次了，从而减少了多次在同步情况下进行判断所浪费的时间。

　　“双重检查加锁”机制的实现会使用关键字volatile，它的意思是：被volatile修饰的变量的值，将不会被本地线程缓存，所有对该变量的读写都是直接操作共享内存，从而确保多个线程能正确的处理该变量。

　　注意：在java1.4及以前版本中，很多JVM对于volatile关键字的实现的问题，会导致“双重检查加锁”的失败，因此“双重检查加锁”机制只只能用在java5及以上的版本。  

---

#####4. 静态内部类
######　　*什么是类级内部类？*  
　　简单点说，类级内部类指的是，有static修饰的成员式内部类。如果没有static修饰的成员式内部类被称为对象级内部类。

　　类级内部类相当于其外部类的static成分，它的对象与外部类对象间不存在依赖关系，因此可直接创建。而对象级内部类的实例，是绑定在外部对象实例中的。

　　类级内部类中，可以定义静态的方法。在静态方法中只能够引用外部类中的静态成员方法或者成员变量。

　　类级内部类相当于其外部类的成员，只有在第一次被使用的时候才被会装载。  


###### 　　*多线程缺省同步锁的知识*
  
　　大家都知道，在多线程开发中，为了解决并发问题，主要是通过使用synchronized来加互斥锁进行同步控制。但是在某些情况中，JVM已经隐含地为您执行了同步，这些情况下就不用自己再来进行同步控制了。这些情况包括：

　　1.由静态初始化器（在静态字段上或static{}块中的初始化器）初始化数据时

　　2.访问final字段时

　　3.在创建线程之前创建对象时

　　4.线程可以看见它将要处理的对象时


```java
public class Singleton {
    
    private Singleton(){}
    /**
     *    类级的内部类，也就是静态的成员式内部类，该内部类的实例与外部类的实例
     *    没有绑定关系，而且只有被调用到时才会装载，从而实现了延迟加载。
     */
    private static class SingletonHolder{
        /**
         * 静态初始化器，由JVM来保证线程安全
         */
        private static Singleton instance = new Singleton();
    }
    
    public static Singleton getInstance(){
        return SingletonHolder.instance;
    }
}
```

　　要想很简单地实现线程安全，可以采用静态初始化器的方式，它可以由JVM来保证线程的安全性。比如前面的饿汉式实现方式。但是这样一来，不是会浪费一定的空间吗？因为这种实现方式，会在类装载的时候就初始化对象，不管你需不需要。

　　如果现在有一种方法能够让类装载的时候不去初始化对象，那不就解决问题了？一种可行的方式就是采用类级内部类，在这个类级内部类里面去创建对象实例。这样一来，只要不使用到这个类级内部类，那就不会创建对象实例，从而同时实现延迟加载和线程安全。  

---

#####5. 枚举 Enum
```java
public enum Singleton {
    /**
     * 定义一个枚举的元素，它就代表了Singleton的一个实例。
     */
    
    uniqueInstance;
    
    /**
     * 单例可以有自己的操作
     */
    public void singletonOperation(){
        //功能处理
    }
}
```
　　我们可以通过EasySingleton.INSTANCE来访问实例，这比调用getInstance()方法简单多了。创建枚举默认就是线程安全的，所以不需要担心double checked locking，而且还能防止反序列化导致重新创建新的对象。

---