##Java多进程----ThreadPoolExecutor实现机制解析
###一、相关概念
ThreadPoolExecutor是jdk内置线程池的一个实现，基本上大部分情况都会使用这个线程池完成并发操作，下面是涉及到ThreadPoolExecutor的相关概念。

1. ThreadPoolExecutor： 这是线程池实现类，会动态创建多个线程，并发执行提交的多个任务；

2. Worker： 是个Runnable实现类来的，内部会创建一个线程，一直循环不断执行任务，所以可以认为一个Worker就是一个工作线程;

3. corePoolSize： 当池中总线程数<corePoolSize时，提交一个任务创建一个新线程，而不管已经存在的线程是不是闲着，通常情况下，一旦线程数达到了corePoolSize，那么池中总线程数是不会跌破到corePoolSize以下的。(除非 allowCoreThreadTimeOut=true并且keepAliveTime>0)；

4. maximumPoolSize： 当池中线程数达到了corePoolSize，这时候新提交的任务就会放入等待队列中，一般情况下，这些任务会被前面创建的 corePoolSize个线程执行。当任务提交速度过快，队列满了，这时候，如果当前总线程数<maximumPoolSize，那么线程池会创建一个新的线程来执行新提交的任务，否则根据策略放弃任务；

5. keepAliveTime：存活时间，分两种情况： (1)allowCoreThreadTimeOut=true，所有线程，一旦创建后，在keepAliveTime时间内，如果没有任务可以执行，则该线程会退出并销毁，这样的好处是系统不忙时可以回收线程资源；(2)allowCoreThreadTimeOut=false，如果总线程数<=corePoolSize，那么这些线程是不会退出的，他们会一直不断的等待任务并执行，哪怕当前没有任务，但如果线程数>corePoolSize，而且一旦一个线程闲的时间超过 keepAliveTime则会退出，但一旦降低到corePoolSize，则不会再退出了。

6. allowCoreThreadTimeOut： 用于决定是否在系统闲时可以逐步回收所有的线程，如果为allowCoreThreadTimeOut=true,必须结合keepAliveTime一起使用，用于决定当线程数<corePoolSize时，是否要回收这些线程。

7. workQueue：这是一个阻塞队列，当线程数>=corePoolSize，这时候提交的任务将会放入阻塞队列中，如果阻塞队列是无界的，那么总的线程数是不可能>corePoolSize的，即maximumPoolSize属性就是无用的；如果阻塞队列是有界的，而且未满，则任务入队，否则根据maximumPoolSize的值判断是要新建线程执行新任务或者是根据策略丢弃任务。

通过下面演示，熟悉上面相关概念： 
![ThreadPoolExecutor演示状态图](https://github.com/Nick-Cheung/Image/blob/master/ThreadPoolExecutor/ThreadPoolExecutor%E6%89%A7%E8%A1%8C%E7%8A%B6%E6%80%81%E5%9B%BE.png?raw=true) 

###二、ThreadPoolExecutor状态

#####ThreadPoolExecutor的五个状态

线程池有5个状态，分别是：
- RUNNING：可以接受新的任务，也可以处理阻塞队列里的任务
- SHUTDOWN：不接受新的任务，但是可以处理阻塞队列里的任务
- STOP：不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
- TIDYING：过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
- TERMINATED：终止状态。terminated方法调用完成以后的状态  


#####状态的转换
状态之间可以进行转换：

- RUNNING -> SHUTDOWN：手动调用shutdown方法，或者ThreadPoolExecutor要被GC回收的时候调用finalize方法，finalize方法内部也会调用shutdown方法

- (RUNNING or SHUTDOWN) -> STOP：调用shutdownNow方法

- SHUTDOWN -> TIDYING：当队列和线程池都为空的时候

- STOP -> TIDYING：当线程池为空的时候

- TIDYING -> TERMINATED：terminated方法调用完成之后


#####状态的表示
 在ThreadPoolExecutor，利用AtomicInteger[(一个提供原子操作的Integer的类)](http://blog.csdn.net/sunnydogzhou/article/details/6564396)保存状态和线程数，整型中32位的前3位用来表示线程池状态，后3位表示线程池中有效的线程数。

// 前3位表示状态，所有线程数占29位  
private static final int COUNT_BITS = Integer.SIZE - 3;  

线程池容量大小为 1 << 29 - 1 = 00011111111111111111111111111111(二进制)，代码如下  
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;  

RUNNING状态 -1 << 29 = 11111111111111111111111111111111 << 29 = 11100000000000000000000000000000(前3位为111)：  
private static final int RUNNING    = -1 << COUNT_BITS;  

SHUTDOWN状态 0 << 29 = 00000000000000000000000000000000 << 29 = 00000000000000000000000000000000(前3位为000)：  
private static final int SHUTDOWN   =  0 << COUNT_BITS;  

STOP状态 1 << 29 = 00000000000000000000000000000001 << 29 = 00100000000000000000000000000000(前3位为001)：  
private static final int STOP       =  1 << COUNT_BITS;  

TIDYING状态 2 << 29 = 00000000000000000000000000000010 << 29 = 01000000000000000000000000000000(前3位为010)：  
private static final int TIDYING    =  2 << COUNT_BITS;  

TERMINATED状态 3 << 29 = 00000000000000000000000000000011 << 29 = 01100000000000000000000000000000(前3位为011)：  
private static final int TERMINATED =  3 << COUNT_BITS;      

清楚状态位之后，下面是获得状态和线程数的内部方法：

```java

    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

	// 得到线程数，也就是后29位的数字。 直接跟CAPACITY做一个与操作即可，CAPACITY就是的值就 1 << 29 - 1 = 00011111111111111111111111111111。 与操作的话前面3位肯定为0，相当于直接取后29位的值
	private static int workerCountOf(int c)  { return c & CAPACITY; }

	// 得到状态，CAPACITY的非操作得到的二进制位11100000000000000000000000000000，然后做在一个与操作，相当于直接取前3位的的值
	private static int runStateOf(int c)     { return c & ~CAPACITY; }

	// 或操作。相当于更新数量和状态两个操作
	private static int ctlOf(int rs, int wc) { return rs | wc; }
```

###三、源码分析
根据javadoc中关于ThreadPoolExecutor类的描述可知。ThreadPoolExecutor的实现主要依靠两个数据结构：

1. 线程池
2. 任务队列

任务队列使用的数据结构比较容易想到，可以采用实现了[java.util.concurrent.BlockingQueue](http://www.cnblogs.com/jackyuj/archive/2010/11/24/1886553.html)接口的类。 
线程池该怎么实现才能让线程池里的任务持续执行一个接一个的任务呢？

#### ThreadPoolExecutor类
```java
public class ThreadPoolExecutor extends AbstractExecutorService {

    ...

    /**
     * Set containing all worker threads in pool. Accessed only when
     * holding mainLock.
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();

    ...    

    /**
     * The queue used for holding tasks and handing off to worker
     * threads.  We do not require that workQueue.poll() returning
     * null necessarily means that workQueue.isEmpty(), so rely
     * solely on isEmpty to see if the queue is empty (which we must
     * do for example when deciding whether to transition from
     * SHUTDOWN to TIDYING).  This accommodates special-purpose
     * queues such as DelayQueues for which poll() is allowed to
     * return null even if it may later return non-null when delays
     * expire.
     */
    private final BlockingQueue<Runnable> workQueue;

    ...

}

```
如代码中的注释所说，workers就是存放工作线程的线程池，就是一个简单的HashSet。那么，关键信息一定是藏在这个Worker类里了。

####Worker类
```java
private final class Worker    
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        // 使用ThreadFactory构造Thread，这个构造的Thread内部的Runnable就是本身，也就是Worker。所以得到Worker的thread并start的时候，会执行Worker的run方法，也就是执行ThreadPoolExecutor的runWorker方法
        setState(-1);  //把状态位设置成-1，这样任何线程都不能得到Worker的锁，除非调用了unlock方法。这个unlock方法会在runWorker方法中一开始就调用，这是为了确保Worker构造出来之后，没有任何线程能够得到它的锁，除非调用了runWorker之后，其他线程才能获得Worker的锁
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

Worker是ThreadPoolExecutor的内部类，成员变量thread就是实际执行任务的线程。这个thread不直接执行用户提交的任务，它执行的任务就是它所在的Worker对象。 Worker对象的run()方法调用了ThreadPoolExecutor.runWorker(Worker w)方法。

####ThreadPoolExecutor.runWorker(Worker w)
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); // 得到当前线程
    Runnable task = w.firstTask; // 得到Worker中的任务task，也就是用户传入的task
    w.firstTask = null; // 将Worker中的任务置空
    w.unlock(); // allow interrupts。 
    boolean completedAbruptly = true;
    try {
        // 如果worker中的任务不为空，继续知否，否则使用getTask获得任务。一直死循环，除非得到的任务为空才退出
        while (task != null || (task = getTask()) != null) {
            w.lock();  // 如果拿到了任务，给自己上锁，表示当前Worker已经要开始执行任务了，已经不是闲置Worker(闲置Worker的解释请看下面的线程池关闭)
            // 在执行任务之前先做一些处理。 1. 如果线程池已经处于STOP状态并且当前线程没有被中断，中断线程 2. 如果线程池还处于RUNNING或SHUTDOWN状态，并且当前线程已经被中断了，重新检查一下线程池状态，如果处于STOP状态并且没有被中断，那么中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task); // 任务执行前需要做什么，ThreadPoolExecutor是个空实现
                Throwable thrown = null;
                try {
                    task.run(); // 真正的开始执行任务，调用的是run方法，而不是start方法。这里run的时候可能会被中断，比如线程池调用了shutdownNow方法
                } catch (RuntimeException x) { // 任务执行发生的异常全部抛出，不在runWorker中处理
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown); // 任务执行结束需要做什么，ThreadPoolExecutor是个空实现
                }
            } finally {
                task = null;
                w.completedTasks++; // 记录执行任务的个数
                w.unlock(); // 执行完任务之后，解锁，Worker变成闲置Worker
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly); // 回收Worker方法
    }
}
```
程序的大致逻辑就是在firstTask或getTask()返回方法不为空的情况下执行task.run()。这里的getTask()方法就是从用户任务队列workQueue获取任务的那个方法。

我们看一下getTask方法是如何获得任务的：
####ThreadPoolExecutor.getTask()
```java
// 如果发生了以下四件事中的任意一件，那么Worker需要被回收：
// 1. Worker个数比线程池最大大小要大
// 2. 线程池处于STOP状态
// 3. 线程池处于SHUTDOWN状态并且阻塞队列为空
// 4. 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)
private Runnable getTask() {
    boolean timedOut = false; // 如果使用超时时间并且也没有拿到任务的标识

    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 如果线程池是SHUTDOWN状态并且阻塞队列为空的话，worker数量减一，直接返回null(SHUTDOWN状态还会处理阻塞队列任务，但是阻塞队列为空的话就结束了)，如果线程池是STOP状态的话，worker数量建议，直接返回null(STOP状态不处理阻塞队列任务)[方法一开始注释的2，3两点，返回null，开始Worker回收]
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        boolean timed;      // 标记从队列中取任务时是否设置超时时间，如果为true说明这个worker可能需要回收，为false的话这个worker会一直存在，并且阻塞当前线程等待阻塞队列中有数据

        for (;;) {
            int wc = workerCountOf(c); // 得到当前线程池Worker个数
            // allowCoreThreadTimeOut属性默认为false，表示线程池中的核心线程在闲置状态下还保留在池中；如果是true表示核心线程使用keepAliveTime这个参数来作为超时时间
            // 如果worker数量比基本大小要大的话，timed就为true，需要进行回收worker
            timed = allowCoreThreadTimeOut || wc > corePoolSize; 

            if (wc <= maximumPoolSize && ! (timedOut && timed)) // 方法一开始注释的1，4两点，会进行下一步worker数量减一
                break;
            if (compareAndDecrementWorkerCount(c)) // worker数量减一，返回null，之后会进行Worker回收工作
                return null;
            c = ctl.get();  // 重新检查线程池状态
            if (runStateOf(c) != rs) // 线程池状态改变的话重新开始外部循环，否则继续内部循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }

        try {
            // 如果需要设置超时时间，使用poll方法，否则使用take方法一直阻塞等待阻塞队列新进数据
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false; // 闲置Worker被中断
        }
    }
}
```

Worker类的执行逻辑大致就是这样了。那么ThreadPoolExecutor是如何新建和启动这些Worker类的呢？ 

来看一下我们提交任务时使用的ThreadPoolExecutor.execute(Runnable command)方法。
####ThreadPoolExecutor.execute(Runable command)
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {   // 第一个步骤，满足线程池中的线程大小比基本大小要小
        if (addWorker(command, true)) // addWorker方法第二个参数true表示使用基本大小
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) { // 第二个步骤，线程池的线程大小比基本大小要大，并且线程池还在RUNNING状态，阻塞队列也没满的情况，加到阻塞队列里
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command)) // 虽然满足了第二个步骤，但是这个时候可能突然线程池关闭了，所以再做一层判断
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false)) // 第三个步骤，直接使用线程池最大大小。addWorker方法第二个参数false表示使用最大大小
        reject(command);
}
```
除去处理ThreadPoolExecutor对象状态的代码，最关键的两段代码就是workQueue.offer(command)和addWorker(command, true)。 
workQueue.offer(command)是将任务加入队列；新建和启动Worker对象的代码就是在addWorker(command, true)里了。

####addWorker(Runnable firstTask, boolean core)
```java
// 两个参数，firstTask表示需要跑的任务。boolean类型的core参数为true的话表示使用线程池的基本大小，为false使用线程池最大大小
// 返回值是boolean类型，true表示新任务被接收了，并且执行了。否则是false
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c); // 线程池当前状态

        // 这个判断转换成 rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty)。 
        // 概括为3个条件：
        // 1. 线程池不在RUNNING状态并且状态是STOP、TIDYING或TERMINATED中的任意一种状态

        // 2. 线程池不在RUNNING状态，线程池接受了新的任务 

        // 3. 线程池不在RUNNING状态，阻塞队列为空。  满足这3个条件中的任意一个的话，拒绝执行任务

        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c); // 线程池线程个数
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) // 如果线程池线程数量超过线程池最大容量或者线程数量超过了基本大小(core参数为true，core参数为false的话判断超过最大大小)
                return false; // 超过直接返回false
            if (compareAndIncrementWorkerCount(c)) // 没有超过各种大小的话，cas操作线程池线程数量+1，cas成功的话跳出循环
                break retry;
            c = ctl.get();  // 重新检查状态
            if (runStateOf(c) != rs) // 如果状态改变了，重新循环操作
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // 走到这一步说明cas操作成功了，线程池线程数量+1
    boolean workerStarted = false; // 任务是否成功启动标识
    boolean workerAdded = false; // 任务是否添加成功标识
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock; // 得到线程池的可重入锁
        w = new Worker(firstTask); // 基于任务firstTask构造worker
        final Thread t = w.thread; // 使用Worker的属性thread，这个thread是使用ThreadFactory构造出来的
        if (t != null) { // ThreadFactory构造出的Thread有可能是null，做个判断
            mainLock.lock(); // 锁住，防止并发
            try {
                // 在锁住之后再重新检测一下状态
                int c = ctl.get();
                int rs = runStateOf(c);

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) { // 如果线程池在RUNNING状态或者线程池在SHUTDOWN状态并且任务是个null
                    if (t.isAlive()) // 判断线程是否还活着，也就是说线程已经启动并且还没死掉
                        throw new IllegalThreadStateException(); // 如果存在已经启动并且还没死的线程，抛出异常
                    workers.add(w); // worker添加到线程池的workers属性中，是个HashSet
                    int s = workers.size(); // 得到目前线程池中的线程个数
                    if (s > largestPoolSize) // 如果线程池中的线程个数超过了线程池中的最大线程数时，更新一下这个最大线程数
                        largestPoolSize = s;
                    workerAdded = true; // 标识一下任务已经添加成功
                }
            } finally {
                mainLock.unlock(); // 解锁
            }
            if (workerAdded) { // 如果任务添加成功，运行任务，改变一下任务成功启动标识
                t.start(); // 启动线程，这里的t是Worker中的thread属性，所以相当于就是调用了Worker的run方法
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted) // 如果任务启动失败，调用addWorkerFailed方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

这个方法里执行了三个我们关注的操作：

- 新建Worker对象——w = new Worker(firstTask);  
- 将Worker对象加入workers集合——workers.add(w);  
- 启动Worker对象里的thread——t.start();  

####总结
简单概括一下ThreadPoolExecutor的运行过程（不包括线程池大小控制、线程池关闭等逻辑）：

1. ThreadPoolExecutor.execute(Runnable command)提交任务
2. 如果Worker数量未达到上限，新建一个Worker并将command作为Worker的firstTask
3. 如果Worker数量已达到上限，则将command放入workQueue
4. 每个启动了的worker先执行firstTask，然后继续从workQueue获取task来执行

###参考资料
1. [Java多线程-工具篇-BlockingQueue](http://www.cnblogs.com/jackyuj/archive/2010/11/24/1886553.html)
2. [Java的多线程编程模型5--从AtomicInteger开始](http://blog.csdn.net/sunnydogzhou/article/details/6564396)
