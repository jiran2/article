# 整体介绍

ReentrantLock是JUC自带的一种可重入锁

主要有两种锁形式

- 公平锁
- 非公平锁

主要功能有两种

- 加锁
- 释放锁

本质是AQS(同步等待队列)，一个带头结点和尾结点的双向链表

当当前节点完成任务，会唤醒队列里面的下一个可唤醒节点

新的任务到来会放到队列的最后面，等待唤醒

# AQS介绍

## 属性

AQS同步等待队列是一个带头尾节点地址的双向链表

重要几个属性

- state
- Node
- head
- tail
- exclusiveOwnerThread

### state

```java
/**
 * 同步状态变量
 * 用于判断锁是否已经被线程持有
 * 当 state >= 1 表明前锁已经被线程持有
 * 如果是可重入锁每次重新进入state+1
 */ 
private volatile int state;
```

### Node

AQS同步队列存的就是Node节点

```java
Node {
    // 在同步队列中等待的线程等待超时或者被中断,取消继续等待
    static final int CANCELLED = 1;
    // 当前结点表示的线程在释放锁后需要唤醒后续节点的线程
    static final int SIGNAL = -1;

    //等待状态 
    //0-初始状态，激活状态
    //1-CANCELLED 取消
    //-1-SIGNAL 等待激活
    volatile int waitStatus;

    //前一个节点的地址
    volatile Node prev;

    //后一个节点的地址
    volatile Node next;

    //当前节点代表的线程
    volatile Thread thread;
}
```

### head

```java
//AQS队列的头结点
private transient volatile Node head;
```

### tail

```java
//AQS队列的尾结点
private transient volatile Node tail;
```

### exclusiveOwnerThread

```java
//当前持有该锁的线程
private transient Thread exclusiveOwnerThread;
```

# ReentrantLock介绍

主要是讲非公平锁，因为用的最多，后面有时间会补充公平锁

ReentranLock通过构造器设置公平锁和非公平锁

```
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

公平锁和非公平锁的类结构图

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201025151721203.png" alt="image-20201025151721203" style="zoom:50%;" align="left" /><img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201025151949428.png" style="zoom:50%;" align="left" />



















## 非公平锁

**非公平锁有两个地方提现**

1. 新节点在Lock()时会先尝试获取锁
2. 第一次尝试获取锁失败时，会加入到等待队列，这时候会判断是不是前一个节点是不是head节点，如果是的话，会再次尝试获取锁

### 加锁

1、调用AQS里面通用加锁逻辑

```java
public void lock() {
   sync.acquire(1);
}
```

2、尝试获取锁或者加入到等待队列，否则中断当前线程

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

尝试加锁【tryAcquire(arg)】

```java
/**
 * 1、获取当前线程信息
 * 2、获取当前锁的state信息
 *    如果state==0代表锁没有被其他线程持有
 *       通过CAS设置state==1
 *       设置锁的持有线程为当前线程
 *       返回获取锁成功
 *    如果state!=0代表锁已经被线程持有，如果锁的的持有线程是当前线程
 *       设置state+1
 *       返回获取锁成功
 * 3、返回获取锁失败
 **/
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

获取锁失败，尝试加入到等待队列【addWaiter(Node.EXCLUSIVE)】

```java
//传入Node模式：static final Node EXCLUSIVE = null , 代表独占模式
private Node addWaiter(Node mode) {
    //将当前争夺锁失败的线程包装为Node，并传入null
    Node node = new Node(mode);
    //注意这里有个死循环
    for (;;) {
        //AbstractQueuedSynchronizer#tail：AQS中维护一个队尾节点Node，初始为null
        Node oldTail = tail;
        
        //同步队列中存在其他线程在排队
        if (oldTail != null) {
            //当前线程对应的Node设置prev指向原来的tail，多线程争夺锁失败时，每个待入队线程会将prev指向为队列中的tail节点；队列初始后将prev指向当前占用锁的线程
            node.setPrevRelaxed(oldTail);
            
            //多线程下，当前线程将tail节点cas运算设置为当前node节点。如果队列只存在自身一个元素，next指向的还是自己，否则指向下一个新Node节点，并返回next指向的节点。
            if (compareAndSetTail(oldTail, node)) {
            
                //oldTail指针指向新的队尾Node，即当前线程
                oldTail.next = node;
                
                //返回新的node节点
                return node;
            }
        } else {
            //初始抢夺锁时，队列尾Node为null时，代表不存在等待线程，需要初始化队列。入队的Node节点head、tail都指向自身，重复上面for循环调用node.setPrevRelaxed，同样将prev指向自身
            initializeSyncQueue();
        }
    }
}


//初始化同步队列，这个队列就是由pre next指针构建的一个双向Node链表
private final void initializeSyncQueue() {
    Node h;
    //初始化时，将head、tail都指向new Node()。即初始化队列时，链表只存在一个节点，这个节点代码当前占用锁的线程
    if (HEAD.compareAndSet(this, null, (h = new Node())))
        tail = h;
}
```

最后挣扎再次尝试获取锁【acquireQueued(addWaiter(Node.EXCLUSIVE), arg))】

```java
/**
 * 从最后一个节点开始查找，依次往上查找上一个节点状态是不是SIGNAL，
 * 如果某个节点上个节点不是SIGNAL，则依次往上找，直到找打SIGNAL
 * 最后的结果就是所有节点的上一个节点状态是SIGNAL，保证上一个节点结束，可以唤醒下一个节点
 **/
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            //获取当前线程Node其prev指针指向的Node p，上面讨论过，队列如果只存在Node，prev拿到的便是自身Node
            final Node p = node.predecessor();
            
            //如果p为链表头结点，即当前线程Node前一个 Node节点为head节点时，尝试抢夺锁(非公平体现)。当队列存在多个Node时，第一个Node的prev指向自身，第二个Node的prev指向第一个Node，那么会出现，这两个Node的head指针都指向head结点，那么他们两个都会抢锁。谁抢成功后，将自身设为head节点，因此head节点实际上代表抢到锁的线程，当它释放锁后会去唤醒后续中断线程，而当它释放锁了新来的Node同样可以抢夺(非公平提现)
            if (p == head && tryAcquire(arg)) {
                //如果抢到锁，将当前线程置为头结点
                setHead(node);
                
                //加锁成功后，将head节点的next指向null
                p.next = null; // help GC
                
                //抢到锁，返回false，无需进入selfInterrupt()中断
                //当interrupted为true时，进入selfInterrupt()开始中断
                return interrupted;
            }
            
            //如果当前线程Node入队时，它的prev节点不是head结点又或者是head节点但是没抢到锁，则应该挂起
            if (shouldParkAfterFailedAcquire(p, node))
                //false | true = false
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}



/**
 * 1、获取当前节点的前一个节点waitStatus
 *    【-1=SIGNAL(当这个节点执行完成可以唤醒下一个节点)  1=CANCELLED(当前节点已经取消任务) 0=默认】
 * 2、为了保证插入的新节点可以被激活，
 *    插入节点的上一个节点必须是SIGNAL，
 *    如果不是就一直往上搜索，直到搜索到SIGNAL状态的节点，
 *    把新节点插入到那个节点后面。
 *
 **/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //初始值为0
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
         //for(;;)第一次将prev节点ws改为Node.SIGNAL(处于该状态时，需要收到一个Signal信号才会unpark)，再次调用时，直接返回true。之后node节点prev不是head节点或者是head但抢锁失败，则挂起。即当当前线程发现它的prev节点处于Node.SIGNAL状态时，会执行挂起并检查是否已经中断。
        return true;
        
    //在AbstractQueuedLongSynchronizer#cancelAcquire取消抢夺锁方法中，node.waitStatus = Node.CANCELLED ，将node waitStatus=Node.CANCELLED(值为1)。
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        //如果node节点的prev节点取消抢夺了，把它的prev节点的prev节点当做它的prev节点，并将node的prev节点的next指向node(建立双向关系)。while循环一直排除之前取消竞争的prev节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         //第一次，将prev节点的waitStatus cas设置为Node.SIGNAL，然后return false
         //如果队列中只存在一个Node，那么将自身的ws设置为Node.SIGNAL，通过for循环再次从acquireQueued方法开始执行再次去抢锁。抢到之后无需中断，没抢到再进入shouldParkAfterFailedAcquire，此时它的ws=-1，进入if(ws == Node.SIGNAL)条件语句
        //如果队列中存在多个Node节点，那么将其prev指向的节点ws设置为Node.SIGNAL，并返回false，重复循环。再从从acquireQueued方法开始执行再次去抢锁。抢到之后无需中断，没抢到再进入shouldParkAfterFailedAcquire，同样会挂起.
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```



### 解锁

1、调用AQS里面释放锁逻辑

```
public void unlock() {
    sync.release(1);
}
```

2、释放锁

```java
public final boolean release(int arg) {
    //释放锁
    //如果tryRelease返回true，代表当前线程完全释放锁
    if (tryRelease(arg)) {
        //拿到head节点
        Node h = head;
        //上面加锁阶段，acquireQueued调用时，ws默认为0，此时并未挂起。如果这里head节点ws则无需unparkSuccessor(h)，否则进入传入head节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}


protected final boolean tryRelease(int releases) {
    ////state-release，如果release>1代表释放多把锁,只有state=0时才真正放弃占有锁
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    //如果state==0，将AQS内部包装的当前线程置为null，并将free标识置为true，否则仍然返回false并将state-release
    setState(c);
    return free;
}


/**
 * ws = 0  激活状态
 * ws < 0  等待激活状态
 * ws > 0  线程取消
 **/
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    //如果ws为Node.SIGNAL，将node.ws cas运算置为0
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    Node s = node.next;
    
    //如果head节点的next节点==null或者已经cancel，从链表尾从后往前开始遍历得到离head节点最近的可唤醒的Node，作为head节点的next，排除ws=1或者node为null的节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        //只要不是head节点，继续for循环
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    
    //对得到的Node节点执行unpark()解除挂起，相当于发送一个Signal信号让其解除挂起，之后该节点线程拿到锁得到CPU执行权进入代码块，其他线程仍然在队列中中断，等待其释放锁。
    if (s != null)
        LockSupport.unpark(s.thread);
}
```





# 参考知识

1. https://juejin.im/post/6844904040103411725
2. JDK 1.8源码
