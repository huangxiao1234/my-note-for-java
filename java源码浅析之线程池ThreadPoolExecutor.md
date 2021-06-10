### 前言
打算把JUC下常用的几个类源码都看一遍并做记录，今天是线程池ThreadPoolExecutor
### ThreadPoolExecutor
看源码前我会先通过其作用猜测源码的大概流程，带着问题去看源码
那么我们复习一下ThreadPoolExecutor的作用是什么
* 调用excute执行一个task
* task会派给线程池中的线程执行
* 如果线程池所有线程都正在执行任务
* 则将task丢入等待队列
* 待有空闲线程后从队列中拉task继续执行
* 线程池最大的作用就是可以复用线程，并且限制线程数

上述是线程池简单的使用流程，可以思考一下怎么实现上述功能
* 需要设计一个等待队列
	* 对外提供get方法获取队列头部元素，如果队列为空，则获取线程需要阻塞
	* 队列还要提供一个put方法，如果队列已满，则放置线程需要阻塞
	* 当成功放置元素后需通知获取线程可以唤醒
	* 当成功获取元素后需通知线程可以继续放置
	* 队列还需要考虑线程安全，即不能一边取一边放，因此要加锁，可以使用ReentryLock
	* 那么阻塞获取和放置线程可以使用两个Condition的await方法
	* 唤醒获取和放置线程可以使用两个Condition的signal方法
* 还需要一个存放线程的线程池	
	* 可以使用一个Set集合存放线程 
	* 当然在添加线程进入集合时需要考虑线程安全，因此也需要上锁
* 还需要设计一个线程执行task的流程
	* 以最常见的流程为例
	* 我们让线程池是懒加载的，即当有task提交时再创建线程
	* 如果task提交太多，导致创建的线程池早已达到最大限制，则将task放入阻塞队列
	* 当线程池中的线程执行完当前task后，去队列中调用队列的获取方法获取一个task
	* 如果队列为空，那么获取线程会阻塞（当然这里还可以加入阻塞的时间条件）
	* 在这里我希望线程池的线程只负责执行task，而不负责放置与获取task，待会看看源码这部分是怎么设计的

至此，一个线程池的思路梳理完毕，我们其实已经可以自定义一个线程池
那么接下来，我们就跟着自己设计的思路来看看大佬们是怎么实现的吧，这样对比着去学习才能够收获更多

#### 构造函数
```java
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
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
* 构造函数传入的参数都很熟悉了就不介绍
* 注意的是初始化时是没有创建线程的

#### execute
接着我们从常用的执行方法```execute()```来入手分析源码。我会通过带序号的注释对源码进行解析，按顺序看注释即可。
进入源码阅读前先进行一个补充说明
* 官方线程池的实现中，每个工作线程是一个Worker，其继承了Thread
* 因此对线程的管理主要是看对Worker的管理
```java
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
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
         

        int c = ctl.get();
        // 1、判断当前线程数是否小于核心线程数
        if (workerCountOf(c) < corePoolSize) {
        	// 2、小于则添加Worker并运行提交的任务，true表示添加的是核心线程Worker
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 3、如果核心线程数已满，则判断当前是否还是可运行状态
        // 如果是，则把任务提交到阻塞队列尾部
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 4、提交成功后，需要判断会不会线程池不可运行了，如果不可运行则把提交的任务去掉
            // 调用拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 5、如果线程池还能运行，但是线程数却没有了（发生意外全被中断了）
            // 则新建一个线程保证刚刚提交的任务能够执行完成
            // 注意这里的worker新增时不需要携带任务去执行，因此新增后会直接去队列中获取任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 6、如果队列已满，则添加非核心线程来执行溢出队列的任务
        else if (!addWorker(command, false))
        	// 7、如果非核心线程也满了，则拒绝
            reject(command);
    }
```
阅读以上代码自问自答下述问题
* 核心线程什么时候会创建？怎么创建的
	* 当有task提交时会创建
	* ==只要新task来且线程池中核心线程未满则都会创建一个新的核心线程加入线程池==
	* 这意味着当核心线程全部空闲时，有新任务提交时，还是会创建新的核心线程执行任务，直到核心线程满为止
	* 所以线程池一定要保证核心线程数能够被充分使用，不然会造成资源浪费
* 非核心线程什么时候会创建？怎么创建的
	* ==当核心线程满后且提交任务至队列溢出时，对于溢出的任务会启动非核心线程进行执行==
	* 以及线程池仍为可运行状态但线程池中线程数为零时，为了保证刚放入队列的任务能被执行，也会启动一个非核心线程
* 需要注意的是，Worker这个类并没有提供用于区分核心线程与非核心线程的结构，我认为之所以要用核心与非核心来区分主要是为了说明线程在不同阶段执行任务的方式吧
	* 对于核心线程，就是一开始拿到任务立马执行，执行完后去阻塞队列中获取任务继续执行
	* 非核心线程是执行队列溢出的任务
	* 但两者并没有区别，待会看看销毁线程时有没有进行区分

那么```excute（）```只能让我们明白线程是如何被创建的，至于线程如何执行任务，如何从队列中获取任务，以及如何添加worker都还不知道，所以我们接着看

#### addWorker
该方法是添加线程时调用的，我在点开其实现前带着一个问题
* 如何保证添加过程中是线程安全的？原子类吗？还是加锁来保证
* 是用什么数据结构？

```java
   private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 1、判断当前线程池状态是否为可运行
            if (rs >= SHUTDOWN && //非可运行
                ! (rs == SHUTDOWN && //非SHUTDOWN
                   firstTask == null && // 任务不为空
                   ! workQueue.isEmpty())) // 队列不为空
                return false;
			
			// 目的为了占锁
            for (;;) {
                int wc = workerCountOf(c);
                // 2、如果线程池为可运行，则根据core判断需要添加的是核心还是非核心
                // 分别判断是否超出
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 3、通过CAS让线程数加一来保证线程安全，如果CAS成功则退出循环
                // 否则继续尝试
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 4、如果发现运行状态发生了改变则重新进入循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        	// 5、把任务交给一个新的worker,worker会new一个新的线程
            w = new Worker(firstTask);
            // 6、获取到线程，如果待会worker成功加入线程池，则可以执行该线程
            final Thread t = w.thread;
            if (t != null) {
            	// 7、由于线程池要考虑线程安全，这里使用可重入锁来加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
					// 8、加锁成功后，再检查一遍线程池的状态
					// rs<SHUTDOWN表示可运行
                    if (rs < SHUTDOWN ||
                    	// 9、表示在抢到锁后，线程池刚好变成SHUTDOWN还是可以执行的
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 10、虽然workers是一个非线程安全的HashSet
                        // 但由于已经加了锁所以可以直接add
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 11、如果worker成功加入线程池，那么可以start对应线程了
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
可以回答一开始的问题了
* 怎么保证添加过程的线程安全？
	* 先判断是否超出限制，若超出直接return false
	* 否则，先用CAS占位
	* 占位成功后使用ReentryLock上锁，将worker加入workers(HashSet)中
* ==需要思考的是上述方法是否最优？有什么好处？==
	* 我们想想有没有别的方法？
		* 其实可以直接对addWorker整段代码上锁
		* 这样CAS都不用了也能保证线程安全
		* 但显然这么做效率会变低，因为如果线程早满了，每次调用addWorker还得先上锁才能进行判断这显然是不合理的
		* 所以我们应该把判断线程数的逻辑拿出锁的外面，只要超出限制就能立刻返回
	* 但这么做就要考虑几个问题，即如果线程数未满，怎么保证多线程添加后总线程数不会超过限制
		* 你会想，那很简单啊，这时候加锁不就行了
		* 我们思考一下这时候只加锁会有什么问题？
			* 如果核心线程数最大值为100，此时同时有1000个任务提交
			* 那么就会试图创建1000个线程，这1000个线程最终都会获取到锁，并添加入线程池
			* 所以加锁后还要进行一次是否达到最大值的判断，这样就能安全了
			* 这也就是单例模式中最常见的double-check机制，相当于在这里又复习了一遍
	* 那么双重检查机制是否ok？
		* 我们接着思考
			* 还是上面的例子
			* 使用双重检查后会出现的情况是，有900个线程最终注定无法加入workers，但他们仍然要继续阻塞等拿到锁才能知道自己不配
			* 这显然效率也是不够高的，那么还能改进吗？
		* 答案就是像源码一样，先通过CAS进行占坑，如果占坑成功，那么才可以获得添加worker的权力，占位失败则重新进入循环，判断是否超出限制
		* 好处就是，CAS占位效率很高，在很短的时间内100个可以获得添加权力的线程就能够占到锁，剩下900个线程占位失败后会重新判断是否超出限制，此时会发现超出限制，从而退出等待
	* 所以==源码就是这个场景下的最优解==！！太妙了！果然从源码中能学到很多东西
* 线程池workers的数据结构？	
	* HashSet
	* ```private final HashSet<Worker> workers = new HashSet<Worker>();```
* 还要注意的是，worker被成功添加进线程池，才能够启动，如果先启动当然就有问题
* 还有加锁时要```try/finally```保证锁的释放这些就不赘述

#### runWorker
Worker在```start```之后会调用其```run```方法，```run```方法执行的是```runWorker```方法
我们看看线程是怎么获取任务并执行的
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
        	// 1、判断task是否不为空，不为空则直接执行
        	// 为空则试图从阻塞队列中获取task，如果阻塞队列为空则会等待
        	// 如果设置了超时时间那么会在超时后返回null,就能够退出循环
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt

				// 2、判断线程池状态，如果要求停止则在执行前把当前线程中断状态置为真
				// 意味着当发生阻塞时会被中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                	// 3、可以重写该方法自定义实现一些运行前的逻辑
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    // 4、执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
* worker执行任务的大概流程其实很简单，就判断是否有task，没有则去队列中获取，有则执行即可
* 需要思考的是代码的一些实现细节，比如怎么实现STOP或者SHUTDOWN的功能？
	* 作者在类一开始注释中就告诉我们线程池有几个状态
	*    SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks
     *  实现SHUTDOWN很简单，我们在前面的源码分析中已经解析过，只需要在添加任务时判断是否为Running即可
     * STOP除了不接受任务外，还要把正在执行的任务中断，并且停止执行队列中的任务
* 因此这部分的重点应该在于思考如何实现STOP中断功能的，也只有这样才能理解为什么源码一开始就进行了一次```unlock```，以及为什么执行方法时要```lock```自己

那么我们先放着上述问题，去看看线程池怎么SHUTDOWN以及STOP的
#### 中断机制
我们先看看切换SHUTDOWN和STOP状态的源码
先看SHUTDOWN的源码
```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
* 我们可以看到，先修改状态值，然后调用```interruptWorkers()```

那么我们接着看```interruptWorkers()```
我们直接进入```interruptIdleWorkers()```
```java
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
* 逻辑很清晰，目的就是将每个worker的线程中断状态设为True
* 由于是对线程池操作，为了保证线程安全需要上一个线程池的主锁==mainlock==
* 接着遍历每个worker，如果其线程中断状态已经为True则不用操作
* 如果为false则尝试获取该worker的锁，获取到锁后方可更新中断状态
	* 这里要思考一下为什么要获取worker的锁
	* 我们联系前面```runWorker```时，在```task.run()```前必须得获得锁，也就意味着，如果worker的线程正在执行任务，那么当前中断线程是获取不到锁的，tryLock不会阻塞，而是直接失败
	* 也就意味着该方法只能对没有在执行task的线程设置中断状态，并且如果正在运行的线程是不会被设中断状态的，下次还能继续运行
	* 设置中断状态为true也就意味着当线程遇到阻塞时会被中断
	* 该方法相当于实现了一个功能
		* 让还在运作的线程继续运作，其他全部中断
* 那么该方法能够实现SHUTDOWN的要求吗？
	* 可以的，因为不会阻断正在运行的任务

注意了，```t.interrupt()```只是改变了中断状态的值，我们还要去思考改了值后，线程会在哪里被中断，是否能实现SHUTDOWN的功能？
* 那么我们现在可以去思考为什么```runWorker```中一开始就调用```w.unlock```释放锁了
	* 对ReentryLock足够了解的话可以知道，如果A线程进行了```w.lock```，B线程进行```w.unlock```会报```IllegalMonitorStateException```的异常(如下小demo)
	* 熟悉ReentryLock的原理知道为什么哈，就是因为unlock的时候会判断ownerThread是否为lock时设置的线程
	* 所以如果worker刚执行```runWorker```方法，main线程就调用```shutdown```抢先一步加锁并且设置中断，则该worker执行到第一步```w.unlock```时就会被中断了，因为上锁的是main线程，worker线程无法解锁
	* 所以可以把未执行任务的线程中断
* 对于runWorker中一开始w.unlock的操作我们进一步地思考
	* 如果是新建的核心线程，那么意味着其携带着task，该线程被中断后，task随之丢失
	* 而如果task已经在队列中，那么不会受到影响
```java
        ReentrantLock lock = new ReentrantLock();
        new Thread(() -> {
            lock.lock();
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "input thread name").start();
        TimeUnit.SECONDS.sleep(1);
        new Thread(() -> {
            lock.unlock();
        }, "input thread name").start();
```
```java
Exception in thread "input thread name" java.lang.IllegalMonitorStateException
	at java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:151)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1261)
	at java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:457)
	at hx.learn.Check.lambda$main$1(Check.java:51)
	at java.lang.Thread.run(Thread.java:748)
```
那么我们再思考一下线程被修改了中断状态后会在哪里被中断，会带来什么影响呢？
* 我们知道中断只有在阻塞的时候才会中断
* 那什么时候会阻塞？
	* task本身会阻塞，但既然task会阻塞说明已经执行了task，也就意味着中断线程无法修改该线程的中断状态，故task阻塞不会被中断
	* 队列为空，worker进行getTask时会阻塞，那么此时就会被中断
	* 队列满时，提交任务的线程如果拒绝策略为一直等待，那么也会阻塞，也会被中断

至此，SHUTDOWN我们就解决了哈，就剩下STOP了
```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
* 该方法与shutdown类似，也是修改状态值然后中断
* 那么我们看看怎么中断的
```java
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```
* 与前一个中断不同之处在于，这里的中断不管三七二十一都给中断了
* 因此可以实现所有线程都被设置中断状态为真
* 还可以看到STOP执行了```tasks = drainQueue();```的操作，就是把队列中的task拷贝出来，并把队列清空，就不具体展开了，因此我们是能够获取队列未执行的任务的

那么我们思考一下所有worker的中断状态都为true后，能否实现STOP关于中断运行时线程的功能？
* 中断未运行的在SHUTDOWN中已经说过了
* 那么运行的能否被中断呢？思考两种情况
	* task已经在执行，并且task阻塞了，由于这时的线程的中断状态为true，所以会被中断
	* task已经在执行，但没有阻塞，那么即时中断状态为true也没法停止
* 所以中断运行线程这个说法并不严谨，应该是中断会发生阻塞的线程

关于线程池的超时部分，异曲同工，就不继续分析了，以及submit方法和阻塞队列的实现也先不分析了，写真么多也有点累了
### 总结
那么以上就是我一边看源码一边思考的记录了，感觉已经能够比较好地掌握线程池，同理，对于连接池等实现应该就更简单了，最后总结一些不是很常见的点
* ctl用32位的前3位表示RunningState，后29位表示WorkerCount
* Worker在执行任务时会给自己上锁，目的是防止中断线程中断自己的执行
* STOP状态只能中断已经执行的有阻塞的task
* 添加线程进入线程池时使用了CAS占位+双重检查的方式保证效率和线程安全