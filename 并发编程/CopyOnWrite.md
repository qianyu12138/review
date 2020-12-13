# CopyOnWrite

### 解决的问题

一个ArrayList在并发环境里，保证线程安全需要在所有操作上加锁，通常情况下加读写锁。

```java
public Object  read() {
    lock.readLock().lock();
    // 对ArrayList读取
    lock.readLock().unlock();
}
public void write() {
    lock.writeLock().lock();
    // 对ArrayList写
    lock.writeLock().unlock();
}
```

而在**读多写少**的环境里，少量的写操作会阻塞大量度操作。

### CopyOnWrite思想

不加锁。

**写数据的时候利用拷贝的副本来执行**

一个ArrayList底层用一个数组来存放数据，当需要写操作时，需要先拷贝一个副本，在副本上进行写操作，期间所有读操作仍然用原来的数组。在副本上写完数据后，将副本的地址赋值给ArrayList，替换掉原来的数据。

#### 保证副本修改后其他读线程能看到-volatile

