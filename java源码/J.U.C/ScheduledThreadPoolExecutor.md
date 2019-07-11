---
title: ScheduledThreadPoolExecutor
Tags: J.U.C
---
## 1. 创建ScheduledThreadPoolExecutor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，实现了ScheduledExecutorService接口，在ThreadPoolExecutor的基础上增加了定时的功能，包括指定延时后执行任务和指定延时后执行任务。ScheduledThreadPoolExecutor的功能与Timer类似，但ScheduledThreadPoolExecutor功能更强大、更灵活。Timer对应的是单个后台线程，而ScheduledThreadPoolExecutor可以多线程执行，所以ScheduledThreadPoolExecutor是比Timer更优的选择。
``` java
    public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {  
        public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
            super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
        }
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从构造函数可以看出ScheduledThreadPoolExecutor还是调用的父类ThreadPoolExecutor的构造方式，其中最大线程数固定为Integer.MAX_VALUE，空闲时间固定为0，工作队列固定为DelayedWorkQueue。DelayedWorkQueue是一个无界队列，故最大线程数并没有什么意义。

#### Executors工具类提供了几种ScheduledThreadPoolExecutor的创建方法：
#### 1. newScheduledThreadPool
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;只指定核心线程数，使用默认的线程工厂以及拒绝策略，最多创建corePoolSize个线程。
``` java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```
#### 2. newSingleThreadScheduledExecutor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;核心线程数为1，使用默认的线程工厂以及拒绝策略，单线程的newScheduledThreadPool。
``` java
    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
```
## 2. 核心思想
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ScheduledThreadPoolExecutor的基本实现依赖于ThreadPoolExecutor，只不过用了特殊的工作队列DelayedWorkQueue（一种基于堆实现的优先队列，类似PriorityQueue），DelayedWorkQueue的队首始终为执行时间最早的一个任务，当worker线程调用DelayedWorkQueue.take()方法时，如果已有leader线程则当前worker线程会成为follower线程进入等待状态（leader/follower多线程网络模型，后面在介绍DelayedWorkQueue时有说明），如果不存在leader线程则当前worker线程会成为leader线程，如果当前时间小于执行时间，则leader线程会阻塞直到当前时间大于等于执行时间才将队首的任务返回，然后leader线程就执行该任务并交出leader权限提拔一个follower线程成为新的leader。如果是周期性任务，则执行后根据执行频率重新计算下次执行时间后再放入DelayedWorkQueue中。
    
## 3. 核心方法
#### 1. schedule方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;schedule有两个重载方法，一个用于执行Runnable任务，一个用于执行Callable任务，command为要执行的任务，delay为延时执行的时间，unit为延时执行的时间单位。二者都是先将任务封装为一个RunnableScheduledFuture对象（实际上就是ScheduledFutureTask），然后放入DelayedWorkQueue队列中，等待时间线程池中的线程从队列中获取任务并执行。Callable与Runnable的区别在于，Callable可以通过调用FutureTask.get()来获取执行结果，不过该方法会阻塞主线程直到获得结果。
``` java
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
```
#### 2. scheduleAtFixedRate方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;scheduleAtFixedRate用于执行固定频率（时间间隔）的任务，command为要执行的任务，initialDelay为第一次执行的延迟时间，period为后面每次执行的时间间隔，unit为时间单位。与schedule方法一样，scheduleAtFixedRate方法也是通过创建一个RunnableScheduledFuture对象（实际上就是ScheduledFutureTask），然后放入DelayedWorkQueue队列中，等待时间线程池中的线程从队列中获取任务并执行。区别在于首次调用scheduleAtFixedRate方法时会在initialDelay时间之后首次执行任务，之后每次根据当前执行时间延迟period时间执行，所以每次执行的时间大致是能确定的。
``` java
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```
#### 3. scheduleWithFixedDelay方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;scheduleWithFixedDelay也是用于执行固定频率的任务，与scheduleAtFixedRate方法不同的是，scheduleWithFixedDelay是在任务执行完成之后的时间加上delay时间延迟执行的，所以每次执行的时间不是确定的（因为不知道任务要执行多久），但间隔是确定的。其他的同scheduleAtFixedRate方法。
``` java
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```
#### 4. delayedExecute方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为ScheduledThreadPoolExecutor是用于执行定时或周期性任务的线程池，所以每次提交任务都是直接放入工作队列而不是直接执行，故每次提交任务会创建空的工作线程直到线程池中的线程数达到核心线程数，然后每个线程都是通过循环调用take()方法去队列中获取任务然后执行。
``` java
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        //如果线程池已停止运行，则执行拒绝策略
        if (isShutdown())
            reject(task);
        else {
            //将任务加入工作队列
            super.getQueue().add(task);
            //再次确认线程池状态，如果是停止状态则将task从工作队列移除，然后取消任务但不中断执行线程
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                // 该方法在ThreadPoolExecutor中实现，是为了确保线程池中至少有一个线程启动，即使corePoolSize为0
                // 在这里是每次添加任务都会创建一个线程，直到线程池中的线程数达到核心线程数
                ensurePrestart();
        }
    }
```
## 4. 重要内部类
#### 1. ScheduledFutureTask
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ScheduledFutureTask继承自FutureTask实现了RunnableScheduledFuture接口，它的time属性代表了该任务的执行时间，sequenceNumber代表了该任务加入ScheduledThreadPoolExecutor的序号，period代表了任务执行的时间间隔。
``` java
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        //添加到ScheduledThreadPoolExecutor中的序号
        private final long sequenceNumber;
        //这个任务要被执行的具体时间
        private long time;
        //任务执行的时间间隔
        private final long period;
        /** The actual task to be re-enqueued by reExecutePeriodic */
        RunnableScheduledFuture<V> outerTask = this;
        //在DelayedWorkQueue中的索引，方便快速取消任务
        int heapIndex;
    }   
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ScheduledFutureTask重写了compareTo方法，首先通过time来比较大小，如果time相同，则跟据sequenceNumber的大小来进行判断。这里主要是为了方便DelayedWorkQueue排序用。

``` java
		public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }
```
#### 2. DelayedWorkQueue
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ScheduledThreadPoolExecutor之所以使用DelayedWorkQueue作为工作队列，是因为定时任务需要优先执行时间靠前的任务，所以ScheduledThreadPoolExecutor就在内部实现了一个线程安全的、阻塞的优先队列。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DelayedWorkQueue是一个基于堆的数据结构,继承自AbstractQueue实现了BlockingQueue接口。内部使用数组实现堆的功能，通过Leader/Follower多线程网络模型防止了动态内存分布及线程间的数据交换。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于DelayedWorkQueue是优先队列，所以用最小堆实现的，即最早要执行的任务放在队列首部。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Leader/Follower多线程网络模型：最多只有一个leader线程用于监听任务，而其他的空闲中的线程（follower）都在等待成为leader，当leader获取到任务之后会首先提拔一个follower线程成为leader，然后自己去执行任务成为processor。
``` java
    static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
        //初始容量
        private static final int INITIAL_CAPACITY = 16;
        //底层用数组实现堆
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
        //使用可重入锁来保证线程安全
        private final ReentrantLock lock = new ReentrantLock();
        //队列大小
        private int size = 0;
        //leader线程，始终为最早要执行任务的执行线程
        private Thread leader = null;
        //与lock结合使用
        private final Condition available = lock.newCondition();
    }
```
##### 1.入列方法（offer） 
``` java
    public boolean offer(Runnable x) {
        if (x == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
        //获取lock实例并加锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = size;
            //如果容量不够，则进行扩容操作
            if (i >= queue.length)
                grow();
            //修改队列大小
            size = i + 1;
            //如果原来队列为空，则直接将任务放入数组索引0处
            if (i == 0) {
                queue[0] = e;
                setIndex(e, 0);
            } 
            //否则根据堆的规则进行排序
            else {
                siftUp(i, e);
            }
            //如果入队的任务在队列首部，则重置leader线程
            if (queue[0] == e) {
                //先将leader线程置为空
                leader = null;
                //再重新提拔一个follower成为leader
                available.signal();
            }
        } finally {
            //释放锁
            lock.unlock();
        }
        return true;
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所有的入列方法都是调用这一个方法。首先判断队列容量是否足够，不够则进行扩容；然后根据最小堆的规则进行排序插入操作；最后判断加入的任务是否具有最高优先级，是则重置leader线程用于执行插入的任务。整个过程通过ReentrantLock加锁来保证线程安全。这里涉及了几个重要的方法，扩容方法grow,排序插入方法siftUp。

##### 2. 扩容方法（grow）
``` java
    private void grow() {
        int oldCapacity = queue.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
        if (newCapacity < 0) // overflow
            newCapacity = Integer.MAX_VALUE;
        queue = Arrays.copyOf(queue, newCapacity);
    }   
```

<html>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;与ArrayList的扩容方法很相似，DelayedWorkQueue容量不够时会自动扩容，每次扩容50%，但是不能超过Integer最大值，通过Arrays.copyOf进行扩容，实际上底层调用System.arraycopy()达到高效扩容的目的。
</html>

##### 3. 排序插入方法（siftUp）
``` java
    private void siftUp(int k, RunnableScheduledFuture<?> key) {
        while (k > 0) {
            //取k位置的父节点索引
            int parent = (k - 1) >>> 1;
            //取父节点e
            RunnableScheduledFuture<?> e = queue[parent];
            //如果key比父节点大，则满足最小堆规则，停止循环
            if (key.compareTo(e) >= 0)
                break;
            //如果父节点较大，则把父节点位置与key交换
            queue[k] = e;
            setIndex(e, k);
            k = parent;
        }
        //通过循环确定了key的索引位置，将key插入该位置
        queue[k] = key;
        setIndex(key, k);
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每次先将插入的任务放在队尾，然后通过比较大小确定最终位置。因为ScheduledFutureTask已经根据时间大小重写了compareTo()方法，所以这里直接通过compareTo()方法比较大小来进行排序。这里需要明白最小堆的规则，需要满足2个条件：1.是完全二叉树，2.父节点的值不能小于子节点的值。最小堆在数组中的表示方法如下：

```    
    // 对于n位置的节点来说：
    int left = 2 * n + 1; // 左子节点
    int right = 2 * n + 2; // 右子节点
    int parent = (n - 1) / 2; // 父节点，当然n要大于0，根节点是没有父节点的
```
##### 4. 等待获取队首方法（take），实现任务调度的核心
``` java
    public RunnableScheduledFuture<?> take() throws InterruptedException {
        //获取lock实例并加锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                //获取队首任务
                RunnableScheduledFuture<?> first = queue[0];
                //如果队首任务为空，进入等待状态
                if (first == null)
                    available.await();
                else {
                    //获取队首任务的延时数
                    long delay = first.getDelay(NANOSECONDS);
                    //延时数小于等于0；说明任务已达调度时间，取出队列交.
                    给leader线程去执行。
                    if (delay <= 0)
                        return finishPoll(first);
                    //任务还未到执行时间，分情况进入等待状态
                    first = null; // don't retain ref while waiting
                    //如果leader线程已存在，则进入等待状态成为follower，等待提拔为leader线程
                    if (leader != null)
                        available.await();
                    else {
                        //如果leader线程不存在，则将当前线程提拔为leader线程
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            //进入等待状态直到达到任务执行时间
                            available.awaitNanos(delay);
                        } finally {
                            //到达执行时间，leader线程去执行任务，成为processor
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            //leader线程已成为processor，如果队首还有任务，则重新提拔一个follower线程成为leader线程
            if (leader == null && queue[0] != null)
                available.signal();
            //释放锁
            lock.unlock();
        }
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;take()方法充分体现了leader/follower多线程网络模型的思想以及ReentrantLock与Condition的配合运用，也是ScheduledThreadPoolExecutor实现任务调度的关键所在。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先通过ReentrantLock加锁，然后去取队首的任务（队首的任务由leader线程执行），如果已有leader线程则成为follower线程等待提拔为leader线程，如果没有leader线程则直接晋升为leader线程，等待达到执行时间后通过finishPoll()方法取出任务并执行，同时交出leader权限，提拔一个follower线程成为新的leader线程来执行下一个任务。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中的follower线程等待通过Condition.await()方法实现，leader线程的等待通过Condition.awaitNanos()实现，提拔一个follower线程成为新的leader线程通过Condition.signal()实现。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DelayedWorkQueue出列的方法还有poll方法，不过线程池只用到了take()方法，take()方法中还有一个出列动作的方法finishPoll()。

##### 5. 出列方法（finishPoll）
``` java
    private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
        //将队列大小-1并赋值给s
        int s = --size;
        //取队尾任务x
        RunnableScheduledFuture<?> x = queue[s];
        //将队尾任务置为空
        queue[s] = null;
        //如果队尾和队首不是同一个任务，则进行移除排序
        if (s != 0)
            siftDown(0, x);
        //将f在堆中的索引设为-1代表已取出，然后将任务返回
        setIndex(f, -1);
        return f;
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;take()方法会阻塞线程直到任务达到执行时间，达到执行时间后就会调用finishPoll()方法将任务取出队列。取出的思路是将队尾的任务放到队首，然后通过siftDown()进行重排序。

##### 6. 移除排序方法（siftDown）
``` java
    private void siftDown(int k, RunnableScheduledFuture<?> key) {
        //取队列长度的一半
        int half = size >>> 1;
        //通过循环比较保证父节点不大于子节点
        while (k < half) {
            //取k的左子节点作为child
            int child = (k << 1) + 1;
            RunnableScheduledFuture<?> c = queue[child];
            //取k的右子节点
            int right = child + 1;
            //如果右子节点在队列中并且左子节点大于右子节点，则取右子节点作为child
            //目的是为了取左右子节点中较小的一个作为child
            if (right < size && c.compareTo(queue[right]) > 0)
                c = queue[child = right];
            //将key与子节点中较小的一个进行比较，如果key较小，说明满足最小堆要求，终止循环
            if (key.compareTo(c) <= 0)
                break;
            //否则将key与child交换位置，然后继续向下检查
            queue[k] = c;
            setIndex(c, k);
            k = child;
        }
        queue[k] = key;
        setIndex(key, k);
    }
```