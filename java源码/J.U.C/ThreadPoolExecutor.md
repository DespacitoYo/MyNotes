---
title: ThreadPoolExecutor
cover_picture: https://despacitoyo.github.io/J.U.C/ThreadPoolExecutor/ThreadPool.jpg
Tags: J.U.C
---
## 1 创建线程池（ThreadPoolExecutor）

ThreadPollExecutor有四个构造函数，但本质上都是调用这一个构造函数。
``` java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        ...
    }
```
**corePoolSize**: 线程池核心线程数量 <br>
**maximumPoolSize**：线程池最大线程数量<br>
**keepAliveTime**：线程空闲时间<br>
**unit**：空闲时间单位<br>
**workQueue**：工作队列，没有空闲线程时新加的任务会放入工作队列中排队等待。工作队列共有四种实现：

    a.ArrayBlockingQueue: 创建固定大小的阻塞队列， 采用的是数组的结构方式
    b.LinkedBlockingQueue: 创建固定大小的阻塞队列，如果为传入参数，则会创建Integer.MaxValue大小的队列
    c.SynchronousQueue: 创建一个不存储元素的阻塞队列，每一个元素的插入都必须等待一个元素的移除操作，不然会一直阻塞。
    d.PriorityBlockingQueue: 一个具有优先级的无限阻塞队列。

**threadFactory**：线程工厂，用于创建线程池中的线程，可以自己实现，默认工厂创建的线程名称为-poolNumber-thread-threadNumber，如：pool-1-thread-10<br>
**handler**: 拒绝策略，当线程池线程达到最大后添加任务不能被执行时的处理策略。拒绝策略有4种实现，当然也可以自己实现（继承RejectExecutionHandle）。

    a. AbortPolicy：默认拒绝策略，丢弃这个任务并抛出RejectedExecutionException异常
    b. DiscardPolicy：直接丢弃，不做任何处理
    c. DiscardOldestPolicy：将工作队列头部元素丢弃（最老的），然后重新提交任务
    d. CallerRunsPolicy：主线程直接去执行这个任务，不用等待线程池
    
#### Executors工具类提供了几种ThreadpoolExecutor的创建方法：

##### 1. FixedThreadPool
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;FixedThreadPool的corePoolSize和maximumPoolSize都被设置为同一个参数nThreads<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;keepAliveTime被设为0L意味着多余的空闲线程会被立即终止<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;FixedThreadPool使用无界队列LinkedBlockingQueue作为线程池的工作队列（队列容量为Integer.MAX_VALUE）。这意味着当线程池中的线程数达到corePoolSize时，新任务会被放入工作队列，因此线程池中的线程不会超过核心线程数。所以此时maximumPoolSize和keepAliveTime都是无效参数，并且只要线程池在运行中便不会拒绝任务。
``` java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
##### 2. SingleThreadExecutor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SingleThreadExecutor是一个使用单个worker线程的Executor，它的核心线程数和最大线程数都被设为1，相当于单线程的FixedThreadPool。
``` java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
##### 3. CachedThreadPool
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CachedThreadPool是一个会根据需要创建新线程的线程池，它的corePoolSize为0，意味着核心线程为空。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;keepAliveTime被设为60L意味着多余的空闲线程会在空闲60s后被终止。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用没有容量的SynchronousQueue作为工作队列，每次offer操作必须等待另一个take操作，反之亦然，SynchronousQueue就像一个门框（门框上不可以站人）。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;最大线程数为Integer.MAX_VALUE，意味着当主线程提交任务速度高于线程池中的线程处理速度时，会无限制的创建新线程。极端情况下，会因为创建过多的线程耗尽CPU和内存资源。
``` java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
## 2 线程池工作原理

    1. 当一个任务被提交时，线程池会判断核心线程数量是否达到最大，如果没有则会通过线程工厂创建一个新的线程来执行任务；如果达到最大则执行步骤2.
    2. 尝试将任务放入工作队列，如果成功加入工作队列，当有工作线程空闲时会去工作队列中获取一个任务来执行；如果加入失败则执行步骤3.
    3. 判断线程数量是否已达到最大线程数量，如果没有则会通过线程工厂创建一个新的线程来执行任务；如果已达到最大线程数，则执行拒绝策略   
![image](/MyNotes/images/J.U.C/ThreadPool.jpg)
## 3 源码分析

### 3.1 核心静态变量及方法
``` java    
    //高3位表示运行的状态，低29未表示线程池中运行的线程数量。
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //Integer的大小为32，Integer.SIZE - 3 = 29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //1左移29位后减1，高3位全为0，低29位全为1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    //高3位：111 【-1的二进制为32位全为1（取反加1），左移29位后高3位为111】
    private static final int RUNNING    = -1 << COUNT_BITS;
    //高3位：000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //高3位: 001
    private static final int STOP       =  1 << COUNT_BITS;
    //高3位：010
    private static final int TIDYING    =  2 << COUNT_BITS;
    //高3位：011
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    // CAPACITY取反后高3位全为1，低29位全为0，与运算后c的低29位全为0，保留高3位获取runState
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // CAPACITY高3位全为0，低29位全为1，与运算后c的高3位全为0，保留低29位获取线程数量
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```
### 3.2 入口方法（execute）

1. 如果工作线程未达到核心线程数，直接创建一个工作线程来执行任务，创建失败则继续执行下一步。
2. 如果线程池正在运行中并且将任务加入工作队列成功，则进行安全检查，虽然加入工作队列成功，但是保不齐其他调用者或者线程此时会修改线程池状态，那么此时我们就需要将任务再进行必要的移除，这是考虑复杂情况的一种安全机制的保障。
3. 如果加入队列失败，工作线程未达到最大线程数，则创建一个新的线程来执行任务，工作线程已达到最大线程数则执行拒绝策略
``` java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        //高3位表示运行的状态，低29未表示线程池中运行的线程数量。
        int c = ctl.get();
        //根据低29位获取线程数
        if (workerCountOf(c) < corePoolSize) {
            //如果工作线程数量小于核心线程数，尝试创建新的worker线程来执行任务
            if (addWorker(command, true))
                return;
            //如果失败可能是在addWorker时ctl发生改变（核心线程数达到最大），所以重新获取值
            c = ctl.get();
        }
        //如果线程池正在运行中并且将任务加入工作队列成功
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //如果此刻线程池已不在运行并且任务从队列移除成功，执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果此刻线程池还在运行，但是工作线程数量为0，则增加一个空的工作线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果加入队列失败，工作线程未达到最大线程数，则创建一个新的线程来执行任务
        else if (!addWorker(command, false))
            //工作线程已达到最大线程数，执行拒绝策略
            reject(command);
    }
```
### 3.3 增加工作线程（addWorker）

首先通过CAS操作增加工作线程数量，增加成功后实例化一个工作线程用来执行任务，然后将工作线程加入works，因为works实际上是一个非线程安全的容器（HashSet），所以通过可重入锁（ReentrantLock）来保证了线程安全问题，防止多个任务同时提交导致works计算不准确，如果添加成功则将该工作线程启动。
``` java
    private boolean addWorker(Runnable firstTask, boolean core) {
        //continue retry; 可以使retry：后面的代码块重新执行
        //break retry; 可以使retry：后面的代码块终止执行，不管有多少层循环嵌套
        retry:
        for (;;) {
            //高3位表示运行的状态，低29未表示线程池中运行的线程数量。
            int c = ctl.get();
            //获取线程池的运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //获取工作线程数量
                int wc = workerCountOf(c);
                //如果工作线程数超过限制，则返回添加失败
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //通过CAS机制增加工作线程数，成功则直接跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //如果CAS操作失败，则说明其他调用者或者线程修改了线程池数据，需要重新读取
                c = ctl.get();  // Re-read ctl
                //如果当前线程池状态已经发生了改变，则从最外层循环开始重新执行
                if (runStateOf(c) != rs)
                    continue retry;
                // 否则继续当前循环
            }
        }
        //工作线程启动状态
        boolean workerStarted = false;
        //工作线程添加状态
        boolean workerAdded = false;
        //工作线程引用
        Worker w = null;
        try {
            //实例化一个工作线程用于执行任务
            w = new Worker(firstTask);
            //取工作线程中的线程
            final Thread t = w.thread;
            if (t != null) {
                //获取可重入锁
                final ReentrantLock mainLock = this.mainLock;
                //加锁，防止多个线程同时提交任务导致works计算不准确，因为works实际上使用HashSet存储的，非线程安全的
                mainLock.lock();
                try {
                    // 获取线程池运行状态
                    int rs = runStateOf(ctl.get());

                    //如果线程池处于运行状态
                    if (rs < SHUTDOWN ||
                        //或者终止状态并且任务为空
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果该线程已运行，则抛出IllegalThreadStateException异常
                        if (t.isAlive())
                            throw new IllegalThreadStateException();
                        //将新建的工作线程加入works
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //调整工作线程添加状态为true
                        workerAdded = true;
                    }
                } finally {
                    //释放锁
                    mainLock.unlock();
                }
                //如果工作线程已添加，则启动该线程，并将工作线程启动状态改为true
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //如果工作线程未启动，执行添加失败逻辑
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
