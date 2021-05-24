# AQS源码阅读

ReentrantLock同步器实现：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
```

```java
abstract static class Sync extends AbstractQueuedSynchronizer{
```

```java
public abstract class AbstractQueuedSynchronizer{
  private transient volatile Node head;

  private transient volatile Node tail;

  private volatile int state;
  static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        volatile int waitStatus;

        volatile Node prev;

        volatile Node next;

        volatile Thread thread;

        Node nextWaiter;
}
```



非公平锁实现：

```java
static final class NonfairSync extends Sync {
  private static final long serialVersionUID = 7316153563782823691L;

  /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
  final void lock() {
    if (compareAndSetState(0, 1))//如果未被锁定，则更新为1
      setExclusiveOwnerThread(Thread.currentThread());//设置当前独占线程为当前线程
    else
      acquire(1);//获取一个state（将state+1）
  }

  protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
  }
}

```

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&//没能获取到state
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//以独占模式将当前线程以node模式放入队列中
    selfInterrupt();
}

//尝试获取一个state
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {//如果未被锁定
    if (!hasQueuedPredecessors() &&//如果是头节点或队列为空
        compareAndSetState(0, acquires)) {//cas+n个state
      setExclusiveOwnerThread(current);//成功，设置独占线程为当前线程
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {//已被锁定，但其实是自己锁定的
    int nextc = c + acquires;//在原有基础上+n个state
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");//传进来的acquire有问题，释放多了
    setState(nextc);
    return true;
  }
  return false;//没成功，即被其他线程占用
}

final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();//前置节点
      if (p == head && tryAcquire(arg)) {//自己是老二且拿到了state
        setHead(node);//自己变成了head
        p.next = null; // help GC 释放了head
        failed = false;//成功了
        return interrupted;
      }
      if (shouldParkAfterFailedAcquire(p, node) &&//睡前需要定义唤醒操作（即找安全点，需要有前置节点唤醒，总不能一直睡）
          parkAndCheckInterrupt())//调用park阻塞，并返回是否由于中断被取消阻塞（中断或unpark）
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)//如果前置节点已经知道释放后需要通知自己
    return true;
  if (ws > 0) {//前置节点挂掉
    do {
      node.prev = pred = pred.prev;//继续寻找可用的节点，并排在后面
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    //设置前驱节点，需要通知自己
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}

//放到队列尾
private Node addWaiter(Node mode) {
  //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享），ReentrantLock是独占
  Node node = new Node(Thread.currentThread(), mode);
  // Try the fast path of enq; backup to full enq on failure
  Node pred = tail;
  if (pred != null) {
    //有tail，接到tail后面就完了
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  enq(node);
  return node;
}

private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize
      //没有tail，那就直接作为head
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```



![img](./img/721070-20151102145743461-623794326.png)



ReentrantLock unlock

```java
public void unlock() {
  sync.release(1);
}
```

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {//释放，返回是否释放完state
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);//通知后置节点
    return true;
  }
  return false;
}
```

```java
protected final boolean tryRelease(int releases) {
  int c = getState() - releases;
  if (Thread.currentThread() != getExclusiveOwnerThread())//当前线程不是此时的独占线程，不允许释放
    throw new IllegalMonitorStateException();
  boolean free = false;
  if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);//清空独占
  }
  setState(c);
  return free;
}
```





https://www.cnblogs.com/waterystone/p/4920797.html