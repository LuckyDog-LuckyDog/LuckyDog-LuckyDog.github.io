---
title: Java基础-线程
date: 2020-03-25 23:24:14
tags: [线程]
categories: [Java]
---

线程是一个程序内部的一条执行路径 ; 线程做为调度和执行的单位, 每个线程拥有独立的运行栈和程序计数器 , 线程切换的开销小比进程小

<!--more-->

#  线程

> 程序 : 为完成特定任务 , 用某种语言编写的一组指令 . 即指的是一段静态的代码

> 进程 : 程序的一次执行过程或是正在运行的一个程序 . 这说明了资源分配的单位 , 程序在运行时会为每个进程分配不同的内存区域 ,进程可以细分为多个线程 , **每个线程**,有自己独立的 :栈 , 程序计数器 **; 多个线程** , 共享一个进程中的结构 : 方法区和堆

> 线程 : 是一个程序内部的一条执行路径 ; 线程做为调度和执行的单位, 每个线程拥有独立的运行栈和程序计数器 , 线程切换的开销小

> 并行 : 多个CPU同时执行多个任务 , 
> 并发 : 一个CPU(采用调度)同时执行多个任务



## 创建线程的四种方式

### 1 ). 继承Thread类

1. 定义Thread类的子类，并重写该类的run()方法，该run()方法的方法体就代表了线程需要完成的任务,因此把
  run()方法称为线程执行体。
2. 创建Thread子类的实例，即创建了线程对象
3. 调用线程对象的start()方法来启动该线程 

```java
构造方法：
public Thread() :分配一个新的线程对象。
public Thread(String name) :分配一个指定名字的新的线程对象。
public Thread(Runnable target) :分配一个带有指定目标新的线程对象。
public Thread(Runnable target,String name) :分配一个带有指定目标新的线程对象并指定名字。
常用方法：
public String getName() :获取当前线程名称。
public void start() :导致此线程开始执行; Java虚拟机调用此线程的run方法。
public void run() :此线程要执行的任务在此处定义代码。
public static void sleep(long millis) :使当前正在执行的线程以指定的毫秒数暂停（暂时停止执行）。
public static native void yield() : 线程让步 , 释放当前cpu的执行权
public final void join() : 在线程a中调用线程b的join() , 此时线程a进入阻塞状态 , 直到线程b完全执行后,a结束阻塞状态
public static Thread currentThread() :返回对当前正在执行的线程对象的引用。
```

### 2 ).实现Runnable接口

1. 定义Runnable接口的实现类，并重写该接口的run()方法，该run()方法的方法体同样是该线程的线程执行体。
2. 创建Runnable实现类的实例，并以此实例作为Thread对象参数来创建Thread对象，该Thread对象才是真正
  的线程对象。
3. 调用线程对象的start()方法来启动线程。 

> **★Thread类与Runnable接口两种方式的区别**
>
> 一个类继承Thread，则不适合资源共享。但是如果实现了Runable接口的话，则很容易的实现资源共享 
>
> 实现Runnable接口比继承Thread类所具有的优势：
>
> 1. 适合多个相同的程序代码的线程去共享同一个资源。
> 2. 可以避免java中的单继承的局限性。
> 3. 增加程序的健壮性，实现解耦操作，代码可以被多个线程共享，代码和线程独立。
> 4. 线程池只能放入实现Runable或Callable类线程，不能直接放入继承Thread的类。

**★结论 : 开发中优先选择Runnable接口实现**

### 3 ). 实现Callable接口

1. 创建一个实现Callable的实现类
2. 重写Call方法 , 将此线程需要执行的操作声明在Call( )中
3. 创建Callable接口的实现类对象
4. 将实现类对象传递到FutureTask的构造器中 ,创建FutureTask对象
5. 将FutureTask对象做为参数传递Thread类的构造器中 , 创建Thread对象 ,调用start 方法
6. 这里 可以回去Callable中的Call方法的返回值 , FutrueTask对象调用get()可以获取

>   **★多线程实现Callable接口比Runnable接口强大**
>
>   1. call()可以有返回值
>   2. call()可以抛出异常 , 被外面的操作捕获 , 获取异常的信息
>   3. Callable支持泛型

### 4 ). 线程池创建

1. 创建线程池对象。 public static ExecutorService newFixedThreadPool(int nThreads)  
2. 创建Runnable接口子类对象。 
3. 提交Runnable接口子类对象。public Future<?> submit(Runnable task)  
4. 关闭线程池

> 

## 线程的六个生命周期

```java
 public enum State {
        NEW, //新建 创建线程对象,但没有调用start()方法
        RUNNABLE,//可运行 线程已经start()方法,可能在运行代码,也可能没在运行代码
        BLOCKED,//锁阻塞 一个线程视图获取锁,但是没有获取到锁
        WAITING,//无限等待 一个线程在等待另一个线程的唤醒动作.
        TIMED_WAITING, //计时等待 一个线程设置了休眠的时间
        TERMINATED;//被终止 run方法执行完毕,或者线程执行出现异常而没有捕获处理 }
```

![8NCID1.png](https://s1.ax1x.com/2020/03/17/8NCID1.png)



## 线程安全的两种方式

### Synchronized锁

#### 1). 同步代码块

```java
synchronized(锁对象){} 
锁对象: 多条线程想要实现同步,那么必须锁对象一致 , (this必须慎用)
```

#### 2). 同步方法

````java
格式:在返回值类型前面加上 synchronized关键字进行修饰
锁对象:
1) 非静态同步方法:锁对象是this (慎用)
2) 静态同步方法:锁对象是该方法所在类的字节码文件对象 获取方式: 类名.class
````

### Lock锁

```java
//使用lock的实现类ReentrantLock
public void lock() :加同步锁。
public void unlock() :释放同步锁
```

### Synchronized与Lock两者之间的不同

相同 : 都是为了解决线程安全

不同点 : Synchronized机制在执行完同步代码之后 就会自动的释放同步监视器

​		Lock需要手动的启动同步(lock()) , 同时结束时也需要手动实现(unlock())

## 线程间的通讯

wait() : 一旦执行此方法 ,当前线程进入阻塞状态 ,并释放同步监视器

notify() : 一旦执行此方法 , 就会唤醒被wait的一个线程 . 如果是多线程 , 唤醒优先级高的线程

notifyAll() : 一旦执行此方法 , 就会唤醒所有被wait()的线程

注意 : 

1. wait() , notify() , notifyAll () 三个方法必须使用在同步代码块或同步方法中
2. wait() , notify() , notifyAll () 的调用者必须是同步代码块或同步方法中的同步监视器 , 否则就会出现IllegelMonitorStateException异常
3. 为确保所有对象都能充当同步监视器 , 那么wait() , notify() , notifyAll ()三个方法都是定义在java.lang.Object上

> **★sleep() 和wait () 的异同?**
>
> 相同点 : 一旦执行方法 , 都可以使得当前的线程进入阻塞状态
>
> 不同点 : 1) 两个方法声明的位置不同 : thread类中声明了sleep() , Object类中声明wait()
>
> ​		2) 调用的要求不同 : sleep()可以在任何需要的场景下调用 , wait()必须使用同步代码锁
>
> ​		3) 关于是否释放同步监视器 : 如果两个方法都使用在同步代码块或同步方法中 , sleep() 不会释放锁 , 而wait () 会释放锁



# AQS分析(转述+分析)

java有了synchronized关键字为什么还会有 Reentrantlock?

线程在执行的时候会调用start的本地方法 

```java
private native void start0();//说明Jvm的线程==操作系统的线程 , 那么synchronized就会去操作操作系统的函数来使线程进行阻塞 
```

1.6之前 , 线程synchronized会操作系统的函数 ,这样每次进行同步的话 ,CPU就会从用户态-->内核态-->用户态 , 那么程序的效率就会降低 ,于是就有了Reentrantlock

```java
public class LockSupport {   
public static void park() {//停车 ,调用该方法线程就会即时阻塞
        UNSAFE.park(false, 0L);}}
```

![](https://img-blog.csdnimg.cn/20190809170223640.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2phdmFfbHl2ZWU=,size_16,color_FFFFFF,t_70)

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    private final Sync sync;//其中Sync 继承的就是AQS
  
  abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        abstract void lock();}}

    public ReentrantLock() {sync = new NonfairSync(); }//无参构造为非公平锁

    public ReentrantLock(boolean fair) {sync = fair ? new FairSync() : new NonfairSync();}//true为公平锁 , false为非公平锁

    static final class FairSync extends Sync {//非公平锁分析
        private static final long serialVersionUID = -3000897897090466540L;
		//上锁
        final void lock() {acquire(1);}

    public final void acquire(int arg) {
      //交替执行时为false , 线程竞争的时候为true , 加入队列
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//这里持有锁的线程,可能释放锁
            selfInterrupt();}
      
    protected final boolean tryAcquire (int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
      //判断锁的状态
            if (c == 0) {
              //线程在交替执行的时候在JVM上进行判断操作,不切换内核 , 所以轻量锁
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
      //重入锁的来由 , 线程竞争的时候 ,return false
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true; }return false;}}

    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    public final boolean hasQueuedPredecessors() {
        Node t = tail;
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }//交替执行,不产生队列的情况下 , 返回false , 有竞争的时候就会队列

    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);//CAS机制
    }

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    
      
	//线程竞争加入队列
    private Node addWaiter(Node mode) {
      //创建竞争线程的节点
        Node node = new Node(Thread.currentThread(), mode);

        Node pred = tail;
      //第一次这里为空 , 直接执行enq(),创建一个空的节点,头尾均指向他
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;}
      
      //链表队列的维护
          private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t; }}}}
      
      	//Node的构造方法
             Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
      
      //是否真的排队 ,因为此时可能持有锁的线程 释放了锁
          final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {//自旋
                final Node p = node.predecessor();//上一个节点
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
              //加锁失败后,需不需要阻塞
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally { 
            if (failed)
                cancelAcquire(node);
        }
    }
      //获取当前节点的上一个节点
            final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
      
      //获得锁失败时候是否进行park , 也就是阻塞 , 注意这里自旋了一次
          private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//0
        if (ws == Node.SIGNAL)//-1
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);//这里将上一个节点的状态设置为-1
        }
        return false;
    }
      
      //阻塞 ,这里的当前线程状态还是为0  , unfase调用unpark的时候就会唤醒,拿到CPU返回执行
          private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
      ======================================================
     //解锁 
          public void unlock() {sync.release(1); }
      
          public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);return true;} return false;}
      
          protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
      		//更改锁的状态
              protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

      	//传入头结点判断头的下一个节点是否为等待的线程
          private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
            //头部的下一个等待的线程
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
            //如果头的下个线程不为null , 那么唤醒他
        if (s != null)LockSupport.unpark(s.thread); }
      
      //唤醒
          public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
```

其中AQS结构是一个类似于双向的链表的结构

``` java
private transient volatile Node head; //队首
private transient volatile Node tail;//尾
private volatile int state;//锁状态，加锁成功则为1，重入+1 解锁则为0
private transient Thread exclusiveOwnerThread;//持有锁的线程

public class Node{
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;}
```



也就是单线程 - 交替执行 , 和队列无关 -- jdk级别就能解决同步问题

## AQS无竞争

![8dPsnH.png](https://s1.ax1x.com/2020/03/17/8dPsnH.png)

## AQS存在竞争

![8hu7tS.png](https://s1.ax1x.com/2020/03/21/8hu7tS.png)

### 竞争Node结构

![8hKkc9.png](https://s1.ax1x.com/2020/03/21/8hKkc9.png)

## AQS过程

![8huyFO.png](https://s1.ax1x.com/2020/03/21/8huyFO.png)












































