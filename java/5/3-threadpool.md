**Java并发**-线程池ThreadPoolExecutor
=========

前言
------------
Hint:首先，我想说java的线程池真的是很绕，以前一直都感觉新建几个线程一直不退出到底是怎么实现的，后来学习了ThreadPoolExecutor源码。学习源码的过程中，最恶心的其实就是几种状态的转换了，这也是ThreadPoolExecutor的核心。花了将近小一周才大致的弄明白ThreadPoolExecutor的机制，遂记录下来。

线程有多重要
------------
Hint:线程是一个程序员一定会涉及到的一个概念，但是线程的创建和切换都是代价比较大的。所以，我们有没有一个好的方案能做到线程的复用呢？这就涉及到一个概念——线程池。合理的使用线程池能够带来3个很明显的好处：

 1. 降低资源消耗：通过重用已经创建的线程来降低线程创建和销毁的消耗
 2. 提高响应速度：任务到达时不需要等待线程创建就可以立即执行。
 3. 提高线程的可管理性：线程池可以统一管理、分配、调优和监控。

java多线程池-ThreadPoolExecutor
------------
Hint:java的线程池支持主要通过ThreadPoolExecutor来实现，我们使用的ExecutorService的各种线程池策略都是基于ThreadPoolExecutor实现的，所以ThreadPoolExecutor十分重要。要弄明白各种线程池策略，必须先弄明白ThreadPoolExecutor。

- - -

(1) 实现原理
------------

![ThreadPoolExecutor实现流程图](http://img.blog.csdn.net/20171112155127808?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjcxMjkwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 1. 调用ThreadPoolExecutor的execute提交线程，首先检查CorePool，如果CorePool内的线程小于CorePoolSize，新创建线程执行任务。
 2. 如果当前CorePool内的线程大于等于CorePoolSize，那么将线程加入到BlockingQueue。
 3. 如果不能加入BlockingQueue，在小于MaxPoolSize的情况下创建线程执行任务。
 4. 如果线程数大于等于MaxPoolSize，那么执行拒绝策略。（可自定义）

- - -

(2) 线程池的创建
------------

线程池的创建可以通过ThreadPoolExecutor的构造方法实现：

```
/**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters.
 *
 * @param corePoolSize the number of threads to keep in the pool, even
 *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
 * @param maximumPoolSize the maximum number of threads to allow in the
 *        pool
 * @param keepAliveTime when the number of threads is greater than
 *        the core, this is the maximum time that excess idle threads
 *        will wait for new tasks before terminating.
 * @param unit the time unit for the {@code keepAliveTime} argument
 * @param workQueue the queue to use for holding tasks before they are
 *        executed.  This queue will hold only the {@code Runnable}
 *        tasks submitted by the {@code execute} method.
 * @param threadFactory the factory to use when the executor
 *        creates a new thread
 * @param handler the handler to use when execution is blocked
 *        because the thread bounds and queue capacities are reached
 * @throws IllegalArgumentException if one of the following holds:<br>
 *         {@code corePoolSize < 0}<br>
 *         {@code keepAliveTime < 0}<br>
 *         {@code maximumPoolSize <= 0}<br>
 *         {@code maximumPoolSize < corePoolSize}
 * @throws NullPointerException if {@code workQueue}
 *         or {@code threadFactory} or {@code handler} is null
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

具体解释一下上述参数：

 1. corePoolSize 核心线程池大小（运行的task线程最大数）
 2. maximumPoolSize 线程池最大容量大小
 3. keepAliveTime 线程池空闲时，线程存活的时间
 4. TimeUnit keepAliveTime存活时间单位
 5. ThreadFactory 线程工厂（可用于定义log打印的线程名称）
 6. BlockingQueue 一个worker（线程池内部线程）对应的task任务队列
 7. RejectedExecutionHandler 线程拒绝策略（可自定义重写；可以定义为拒绝方法内部自己再提交一次任务不断提交）

- - -

(3) 线程的提交
------------

ThreadPoolExecutor的构造方法如上所示，但是只是做一些参数的初始化，ThreadPoolExecutor被初始化好之后便可以提交线程任务，线程的提交方法主要是execute和submit。这里主要说execute，submit会在后续的博文中分析。

```
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     * 如果当前的线程数小于核心线程池的大小，根据现有的线程作为第一个Worker运行的线程，
     * 新建一个Worker，addWorker自动的检查当前线程池的状态和Worker的数量，
     * 防止线程池在不能添加线程的状态下添加线程
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *  如果线程入队成功，然后还是要进行double-check的，因为线程池在入队之后状态是可能会发生变化的
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     * 
     * 如果task不能入队(队列满了)，这时候尝试增加一个新线程，如果增加失败那么当前的线程池状态变化了或者线程池已经满了
     * 然后拒绝task
     */
    int c = ctl.get();
    //当前的Worker的数量小于核心线程池大小时，新建一个Worker。
    if (workerCountOf(c) < corePoolSize) { 
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))//recheck防止线程池状态的突变，如果突变，那么将reject线程，防止workQueue中增加新线程
            reject(command);
        else if (workerCountOf(recheck) == 0)//上下两个操作都有addWorker的操作，但是如果在workQueue.offer的时候Worker变为0，
                                            //那么将没有Worker执行新的task，所以增加一个Worker.
            addWorker(null, false);
    }
    //如果workQueue满了，那么这时候可能还没到线程池的maxnum，所以尝试增加一个Worker
    else if (!addWorker(command, false))
        reject(command);//如果Worker数量到达上限，那么就拒绝此线程
}
```

这里需要明确几个概念：

 1. Worker和Task的区别，Worker是当前线程池中的线程，而task虽然是runnable，但是并没有真正执行，只是被Worker调用了run方法，后面会看到这部分的实现。
 2. maximumPoolSize和corePoolSize的区别：这个概念很重要，maximumPoolSize为线程池最大容量，也就是说线程池最多能起多少Worker。corePoolSize是核心线程池的大小，当corePoolSize满了时，同时workQueue full（ArrayBolckQueue是可能满的） 那么此时允许新建Worker去处理workQueue中的Task，但是不能超过maximumPoolSize。超过corePoolSize之外的线程会在空闲超时后终止。

- - -

(4) 核心方法：addWorker
--------

Worker的增加和Task的获取以及终止都是在此方法中实现的，也就是这一个方法里面包含了很多东西。在addWorker方法中提到了Status的概念，Status是线程池的核心概念，这里我们先看一段关于status的注释：

```
/**
* 首先ctl是一个原子量，同时它里面包含了两个field，一个是workerCount，另一个是runState
* workerCount表示当前有效的线程数，也就是Worker的数量
* runState表示当前线程池的状态
* The main pool control state, ctl, is an atomic integer packing
* two conceptual fields
*   workerCount, indicating the effective number of threads
*   runState,    indicating whether running, shutting down etc
* 
* 两者是怎么结合的呢？首先workerCount是占据着一个atomic integer的后29位的，而状态占据了前3位
* 所以，workerCount上限是(2^29)-1。
* In order to pack them into one int, we limit workerCount to
* (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
* billion) otherwise representable. If this is ever an issue in
* the future, the variable can be changed to be an AtomicLong,
* and the shift/mask constants below adjusted. But until the need
* arises, this code is a bit faster and simpler using an int.
*
* The workerCount is the number of workers that have been
* permitted to start and not permitted to stop.  The value may be
* transiently different from the actual number of live threads,
* for example when a ThreadFactory fails to create a thread when
* asked, and when exiting threads are still performing
* bookkeeping before terminating. The user-visible pool size is
* reported as the current size of the workers set.
*
* runState是整个线程池的运行生命周期，有如下取值：
*  1. RUNNING：可以新加线程，同时可以处理queue中的线程。
*  2. SHUTDOWN：不增加新线程，但是处理queue中的线程。
*  3.STOP 不增加新线程，同时不处理queue中的线程。
*  4.TIDYING 所有的线程都终止了（queue中），同时workerCount为0，那么此时进入TIDYING
*  5.terminated()方法结束，变为TERMINATED
* The runState provides the main lifecyle control, taking on values:
*
*   RUNNING:  Accept new tasks and process queued tasks
*   SHUTDOWN: Don't accept new tasks, but process queued tasks
*   STOP:     Don't accept new tasks, don't process queued tasks,
*             and interrupt in-progress tasks
*   TIDYING:  All tasks have terminated, workerCount is zero,
*             the thread transitioning to state TIDYING
*             will run the terminated() hook method
*   TERMINATED: terminated() has completed
*
* The numerical order among these values matters, to allow
* ordered comparisons. The runState monotonically increases over
* time, but need not hit each state. The transitions are:
* 状态的转化主要是：
* RUNNING -> SHUTDOWN（调用shutdown()）
*    On invocation of shutdown(), perhaps implicitly in finalize()
* (RUNNING or SHUTDOWN) -> STOP(调用shutdownNow())
*    On invocation of shutdownNow()
* SHUTDOWN -> TIDYING（queue和pool均empty）
*    When both queue and pool are empty
* STOP -> TIDYING（pool empty，此时queue已经为empty）
*    When pool is empty
* TIDYING -> TERMINATED(调用terminated())
*    When the terminated() hook method has completed
*
* Threads waiting in awaitTermination() will return when the
* state reaches TERMINATED.
*
* Detecting the transition from SHUTDOWN to TIDYING is less
* straightforward than you'd like because the queue may become
* empty after non-empty and vice versa during SHUTDOWN state, but
* we can only terminate if, after seeing that it is empty, we see
* that workerCount is 0 (which sometimes entails a recheck -- see
* below).
*/
```

下面是状态的代码：

```
//利用ctl来保证当前线程池的状态和当前的线程的数量。ps：低29位为线程池容量，高3位为线程状态。
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//设定偏移量
private static final int COUNT_BITS = Integer.SIZE - 3;
//确定最大的容量2^29-1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
//几个状态，用Integer的高三位表示
// runState is stored in the high-order bits
//111
private static final int RUNNING    = -1 << COUNT_BITS;
//000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//001
private static final int STOP       =  1 << COUNT_BITS;
//010
private static final int TIDYING    =  2 << COUNT_BITS;
//011
private static final int TERMINATED =  3 << COUNT_BITS;
//获取线程池状态，取前三位
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取当前正在工作的worker,主要是取后面29位
private static int workerCountOf(int c)  { return c & CAPACITY; }
//获取ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

接下来贴上addWorker方法看看：

```
/**
 * Checks if a new worker can be added with respect to current
 * pool state and the given bound (either core or maximum). If so,
 * the worker count is adjusted accordingly, and, if possible, a
 * new worker is created and started, running firstTask as its
 * first task. This method returns false if the pool is stopped or
 * eligible to shut down. It also returns false if the thread
 * factory fails to create a thread when asked.  If the thread
 * creation fails, either due to the thread factory returning
 * null, or due to an exception (typically OutOfMemoryError in
 * Thread.start()), we roll back cleanly.
 *
 * @param firstTask the task the new thread should run first (or
 * null if none). Workers are created with an initial first task
 * (in method execute()) to bypass queuing when there are fewer
 * than corePoolSize threads (in which case we always start one),
 * or when the queue is full (in which case we must bypass queue).
 * Initially idle threads are usually created via
 * prestartCoreThread or to replace other dying workers.
 *
 * @param core if true use corePoolSize as bound, else
 * maximumPoolSize. (A boolean indicator is used here rather than a
 * value to ensure reads of fresh values after checking other pool
 * state).
 * @return true if successful
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        /**
		* rs!=Shutdown || fistTask！=null || workCount.isEmpty
		* 如果当前的线程池的状态>SHUTDOWN 那么拒绝Worker的add 如果=SHUTDOWN
		* 那么此时不能新加入不为null的Task，如果在WorkCount为empty的时候不能加入任何类型的Worker，
		* 如果不为empty可以加入task为null的Worker,增加消费的Worker
		*/
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

addWorker中首先进行了一次线程池状态的检测：

```
int c = ctl.get();
int rs = runStateOf(c);

// Check if queue empty only if necessary.
//判断当前线程池的状态是不是已经shutdown，如果shutdown了拒绝线程加入
//(rs!=SHUTDOWN || first!=null || workQueue.isEmpty())
//如果rs不为SHUTDOWN，此时状态是STOP、TIDYING或TERMINATED，所以此时要拒绝请求
//如果此时状态为SHUTDOWN，而传入一个不为null的线程，那么需要拒绝
//如果状态为SHUTDOWN，同时队列中已经没任务了，那么拒绝掉
if (rs >= SHUTDOWN &&
    ! (rs == SHUTDOWN &&
       firstTask == null &&
       ! workQueue.isEmpty()))
    return false;

```

其实是比较难懂的，主要在线程池状态判断条件这里：

 1. 如果是runing，那么跳过if。
 2. 如果rs>=SHUTDOWN,同时不等于SHUTDOWN，即为SHUTDOWN以上的状态，那么不接受新线程。
 3. 如果rs>=SHUTDOWN，同时等于SHUTDOWN，同时first！=null，那么拒绝新线程，如果first==null，那么可能是新增加线程消耗Queue中的线程。但是同时还要检测workQueue是否isEmpty()，如果为Empty，那么队列已空，不需要增加消耗线程，如果队列没有空那么运行增加first=null的Worker。

从这里是可以看出一些策略的

 1. 首先，在rs>SHUTDOWN时，拒绝一切线程的增加，因为STOP是会终止所有的线程，同时移除Queue中所有的待执行的线程的，所以也不需要增加first=null的Worker了
 2. 其次，在SHUTDOWN状态时，是不能增加first！=null的Worker的，同时即使first=null，但是此时Queue为Empty也是不允许增加Worker的，SHUTDOWN下增加的Worker主要用于消耗Queue中的任务。
 3. SHUTDOWN状态时，是不允许向workQueue中增加线程的，isRunning(c) && workQueue.offer(command) 每次在offer之前都要做状态检测，也就是线程池状态变为>=SHUTDOWN时不允许新线程进入线程池了。

```
for (;;) {
     int wc = workerCountOf(c);
     //如果当前的数量超过了CAPACITY，或者超过了corePoolSize和maximumPoolSize（试core而定）
     if (wc >= CAPACITY ||
         wc >= (core ? corePoolSize : maximumPoolSize))
         return false;
     //CAS尝试增加线程数，如果失败，证明有竞争，那么重新到retry。
     if (compareAndIncrementWorkerCount(c))
         break retry;
     c = ctl.get();  // Re-read ctl
     //判断当前线程池的运行状态
     if (runStateOf(c) != rs)
         continue retry;
     // else CAS failed due to workerCount change; retry inner loop
 }
```
这段代码做了一个兼容，主要是没有到corePoolSize 或maximumPoolSize上限时，那么允许添加线程，CAS增加Worker的数量后，跳出循环。
接下来实例化Worker,实例化Worker其实是很关键的，后面会说。
因为workers是HashSet线程不安全的，那么此时需要加锁，所以mainLock.lock(); 之后重新检查线程池的状态，如果状态不正确，那么减小Worker的数量，为什么tryTerminate（）目前不大清楚。如果状态正常，那么添加Worker到workers。最后：

```
if (runStateOf(ctl.get()) == STOP && ! t.isInterrupted())
         t.interrupt();
```

注释说的很清楚，为了能及时的中断此Worker，因为线程存在未Start的情况，此时是不能响应中断的，如果此时status变为STOP，则不能中断线程。此处用作中断线程之用。

- - -

(5) Worker中的方法
--------

```
/**
* Creates with given first task and thread from ThreadFactory.
* @param firstTask the first task (null if none)
*/
Worker(Runnable firstTask) {
 this.firstTask = firstTask;
 this.thread = getThreadFactory().newThread(this);
}
```
这里可以看出Worker是对firstTask的包装，并且Worker本身就是Runnable的，看上去真心很流氓的感觉~~~
通过ThreadFactory为Worker自己构建一个线程。
因为Worker是Runnable类型的，所以是有run方法的,上面也看到了会调用t.start() 其实就是执行了run方法：

```
/** Delegates main run loop to outer runWorker  */
public void run() {
 runWorker(this);
}
```

调用了runWorker:

```
/**
  * Main worker run loop.  Repeatedly gets tasks from queue and
  * executes them, while coping with a number of issues:
  * 1 Worker可能还是执行一个初始化的task——firstTask。
  *       但是有时也不需要这个初始化的task（可以为null）,只要pool在运行，就会
  *   通过getTask从队列中获取Task，如果返回null，那么worker退出。
  *   另一种就是external抛出异常导致worker退出。
  * 1. We may start out with an initial task, in which case we
  * don't need to get the first one. Otherwise, as long as pool is
  * running, we get tasks from getTask. If it returns null then the
  * worker exits due to changed pool state or configuration
  * parameters.  Other exits result from exception throws in
  * external code, in which case completedAbruptly holds, which
  * usually leads processWorkerExit to replace this thread.
  * 
  * 
  * 2 在运行任何task之前，都需要对worker加锁来防止other pool中断worker。
  *      clearInterruptsForTaskRun保证除了线程池stop，那么现场都没有中断标志
  * 2. Before running any task, the lock is acquired to prevent
  * other pool interrupts while the task is executing, and
  * clearInterruptsForTaskRun called to ensure that unless pool is
  * stopping, this thread does not have its interrupt set.
  *
  * 3. Each task run is preceded by a call to beforeExecute, which
  * might throw an exception, in which case we cause thread to die
  * (breaking loop with completedAbruptly true) without processing
  * the task.
  *
  * 4. Assuming beforeExecute completes normally, we run the task,
  * gathering any of its thrown exceptions to send to
  * afterExecute. We separately handle RuntimeException, Error
  * (both of which the specs guarantee that we trap) and arbitrary
  * Throwables.  Because we cannot rethrow Throwables within
  * Runnable.run, we wrap them within Errors on the way out (to the
  * thread's UncaughtExceptionHandler).  Any thrown exception also
  * conservatively causes thread to die.
  *
  * 5. After task.run completes, we call afterExecute, which may
  * also throw an exception, which will also cause thread to
  * die. According to JLS Sec 14.20, this exception is the one that
  * will be in effect even if task.run throws.
  *
  * The net effect of the exception mechanics is that afterExecute
  * and the thread's UncaughtExceptionHandler have as accurate
  * information as we can provide about any problems encountered by
  * user code.
  *
  * @param w the worker
  */
 final void runWorker(Worker w) {
     Runnable task = w.firstTask;
     w.firstTask = null;
     //标识线程是不是异常终止的
     boolean completedAbruptly = true;
     try {
         //task不为null情况是初始化worker时，如果task为null，则去队列中取线程--->getTask()
         while (task != null || (task = getTask()) != null) {
             w.lock();
             //获取woker的锁，防止线程被其他线程中断
             clearInterruptsForTaskRun();//清楚所有中断标记
             try {
                 beforeExecute(w.thread, task);//线程开始执行之前执行此方法，可以实现Worker未执行退出，本类中未实现
                 Throwable thrown = null;
                 try {
                     task.run();
                 } catch (RuntimeException x) {
                     thrown = x; throw x;
                 } catch (Error x) {
                     thrown = x; throw x;
                 } catch (Throwable x) {
                     thrown = x; throw new Error(x);
                 } finally {
                     afterExecute(task, thrown);//线程执行后执行，可以实现标识Worker异常中断的功能，本类中未实现
                 }
             } finally {
                 task = null;//运行过的task标null
                 w.completedTasks++;
                 w.unlock();
             }
         }
         completedAbruptly = false;
     } finally {
         //处理worker退出的逻辑
         processWorkerExit(w, completedAbruptly);
     }
 }
```

从上面代码可以看出，execute的Task是被“包装 ”了一层，线程启动时是内部调用了Task的run方法。
接下来所有的核心集中在getTask()方法上：

```
/**
  * Performs blocking or timed wait for a task, depending on
  * current configuration settings, or returns null if this worker
  * must exit because of any of:
  * 1. There are more than maximumPoolSize workers (due to
  *    a call to setMaximumPoolSize).
  * 2. The pool is stopped.
  * 3. The pool is shutdown and the queue is empty.
  * 4. This worker timed out waiting for a task, and timed-out
  *    workers are subject to termination (that is,
  *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
  *    both before and after the timed wait.
  *
  * @return task, or null if the worker must exit, in which case
  *         workerCount is decremented
  *         
  *         
  *  队列中获取线程
  */
 private Runnable getTask() {
     boolean timedOut = false; // Did the last poll() time out?

     retry:
     for (;;) {
         int c = ctl.get();
         int rs = runStateOf(c);

         // Check if queue empty only if necessary.
         //当前状态为>stop时，不处理workQueue中的任务，同时减小worker的数量所以返回null，如果为shutdown 同时workQueue已经empty了，同样减小worker数量并返回null
         if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
             decrementWorkerCount();
             return null;
         }

         boolean timed;      // Are workers subject to culling?

         for (;;) {
             int wc = workerCountOf(c);
             timed = allowCoreThreadTimeOut || wc > corePoolSize;

             if (wc <= maximumPoolSize && ! (timedOut && timed))
                 break;
             if (compareAndDecrementWorkerCount(c))
                 return null;
             c = ctl.get();  // Re-read ctl
             if (runStateOf(c) != rs)
                 continue retry;
             // else CAS failed due to workerCount change; retry inner loop
         }

         try {
             Runnable r = timed ?
                 workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                 workQueue.take();
             if (r != null)
                 return r;
             timedOut = true;
         } catch (InterruptedException retry) {
             timedOut = false;
         }
     }
 }
```

这段代码十分关键，首先看几个局部变量：
 1. boolean timedOut = false; 主要是判断后面的poll是否要超时
 2. boolean timed;主要是标识着当前Worker超时是否要退出。wc > corePoolSize时需要减小空闲的Worker数，那么timed为true，但是wc <= corePoolSize时，不能减小核心线程数timed为false。timedOut初始为false，如果timed为true那么使用poll取线程。如果正常返回，那么返回取到的task。如果超时，证明worker空闲，同时worker超过了corePoolSize，需要删除。返回r=null。则 timedOut = true。此时循环到wc <= maximumPoolSize && ! (timedOut && timed)时，减小worker数，并返回null，导致worker退出。如果线程数<= corePoolSize，那么此时调用 workQueue.take()，没有线程获取到时将一直阻塞，知道获取到线程或者中断，关于中断后面Shutdown的时候会说。

至此线程执行过程就分析完了~~~~
