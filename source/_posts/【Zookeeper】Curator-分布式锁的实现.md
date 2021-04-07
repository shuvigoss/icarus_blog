---
title: 【Zookeeper】Curator-分布式锁的实现
date: 2017-03-10 16:10:55
tags: Distributed Lock
categories: Zookeeper
---

[CuratorFramework](http://curator.apache.org/)提供了对于`Zookeeper`client的封装,使调用者能够更方便的使用`Zookeeper`,同时也提供了很多菜谱(recipes),其中的分布式锁是最常用的,那它是怎么基于`Zookeeper`实现的分布式锁呢?

<!--more-->

### simple
由于很多关于Zookeeper分布式锁的实现上都采用创建同一节点,成功则获得锁,失败则等待并且监听这个节点。

这种方式很像`Java`中的`synchronized`关键字,每一次获得锁都是竞争的关系.我大致画了一下这种方式的实现思路.

![](/images/InterProcessMutex-normal.gif)

实现这里就不写了,大部分的实现都是这个思路.

### Curator InterProcessMutex
这里着重讲一下`Curator`的实现:`InterProcessMutex`,首先看一下设计思路.

![](/images/InterProcessMutex.gif)

它的实现思路比较特别

1. 生成以 "lock-" + uuid + sequence 的`EPHEMERAL_SEQUENTIAL`节点.
2. 查询所有lockPath下的节点.
3. 如果查询到当前线程创建的 uuid 节点,表明当前线程已经获得锁或者在锁的等待队列中.如果没有查询到,说明当前`Zookeeper`节点并未同步完数据,跳转到*2*.
4. 对查询所有节点的List进行筛选,如果自己创建的节点并不是List的第0个元素,说明自己未获得锁,那么进行等待并且设置Watcher观察自己的前置节点,如果Watcher被调用,返回*2*重新执行.如果自己创建的节点等于List第0个元素,说明已经获取锁,返回true.

这样做的方式很像`Java`中的`ReentrantLock`,有一个等待队列,保证了线程获取锁的顺序,并且支持重入,但它只有公平锁特性.

>   在我刚读代码的时候有一个疑虑,就是如果你当前线程(进程)创建的自己的临时节点，
>   在获取的时候怎么可能获取不到自己创建的节点呢?这块是由于我当时对Zookeeper
>   的ZAB协议理解并不深.在Leader收到Transcation的proposal时,会以原子广播
>   的形式发给所有Follower,它认为集群中超过半数+1节点返回成功那么就是成功,
>   那么这就存在一个问题就是不在这半数+1节点的集群中数据在这个时间点上会存在
>   暂时的不一致情况,依赖后面的同步数据来保证数据最终的一致性.所以说Zookeeper
>   的一致性是最终一致性.


#### acquire Lock
下面来看代码实现.

``` java
InterProcessMutex lock = new InterProcessMutex(client, "/mutex");
try {
  lock.acquire();
  System.out.println("i got lock");
} catch (Exception e) {
  e.printStackTrace();
}
```
在构造函数里会指定需要加锁的Path,所有加锁线程会在这个Path下创建临时有序节点.

``` java
//这个lock-前缀是为了区分这个是一个锁标示,有可能在指定节点路径下会有其他节点.
//这个是在获取锁节点时的一个过滤条件,非以lock-开头的节点会被认为是无用节点.
private static final String LOCK_NAME = "lock-";
```

``` java
@Override
public void acquire() throws Exception
{
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}

@Override
public boolean acquire(long time, TimeUnit unit) throws Exception
{
    return internalLock(time, unit);
}
```
这2个方法是获取锁的入口方法,提供了超时等待的功能.

``` java
private boolean internalLock(long time, TimeUnit unit) throws Exception
{
    /*
       Note on concurrency: a given lockData instance
       can be only acted on by a single thread so locking isn't necessary
    */

    Thread currentThread = Thread.currentThread();
    //获取当前缓存中是已经获取过锁,这是重入的一个实现,如果已经获取锁,
    //那么对lockCount进行++
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null )
    {
        // re-entering
        lockData.lockCount.incrementAndGet();
        return true;
    }
    //尝试获取锁,如果获得锁,会将线程信息存入threadData并返回true
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);
        return true;
    }

    return false;
}
```

``` java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
{
    //设置一些本地变量,后边依据这些变量进行超时判断,是否获取锁等等.
    final long      startMillis = System.currentTimeMillis();
    final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int             retryCount = 0;

    String          ourPath = null;
    boolean         hasTheLock = false;
    boolean         isDone = false;
    while ( !isDone )
    {
        isDone = true;

        try
        {
            //开始创建节点
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            //根据创建节点进行过滤并且判断是否是第0个元素
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            // gets thrown by StandardLockInternalsDriver when it can't find the lock node
            // this can happen when the session expires, etc. So, if the retry allows, just try it all again
            // 重试,这块在internalLockLoop内部判断ourPath是否存在children List内时会
            // 抛出NoNodeException,这就是解决某一节点同步延迟的情况
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
            {
                isDone = false;
            }
            else
            {
                throw e;
            }
        }
    }

    if ( hasTheLock )
    {
        return ourPath;
    }

    return null;
}
```

``` java
@Override
public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception
{
    String ourPath;
    if ( lockNodeBytes != null )
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, lockNodeBytes);
    }
    else
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
    }
    return ourPath;
}
```
`createsTheLock`就是创建一个临时有序节点,其中的`withProtection()`方法就是创建带有uuid的节点方法,为什么会有这种需求呢?我读了下`ProtectACLCreateModePathAndBytesable`的介绍,这是一种设计上的优化,针对创建一个`SEQUENTIAL`节点,客户端并不能知道你创建的节点到底是什么path,有了这种GUID的设计能够在不查询节点内容的情况下知道自己创建的有序节点是哪个.

``` java
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try
    {
        //这是锁撤销的一个扩展,如果设置了可撤销,那么在需要的时候进行锁节点变更,
        //将节点内容写为"__REVOKE__"就可实现锁撤销
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            //获取Path下所有节点,进行排序和过滤,这里返回的是一个所有锁节点的排序结果
            List<String>        children = getSortedChildren();
            //获得自己锁的uuid
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
            //判断是否获得锁并且需要Watcher的节点名称(也就是前置节点)
            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            //获得锁,即将返回
            if ( predicateResults.getsTheLock() )
            {
                haveTheLock = true;
            }
            else
            {
                //未获得锁,在等待时间内观察前置节点,
                //唤醒的条件是前置节点变更或者等待时间到
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this)
                {
                    try 
                    {
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }

                            wait(millisToWait);
                        }
                        else
                        {
                            wait();
                        }
                    }
                    catch ( KeeperException.NoNodeException e ) 
                    {
                        // it has been deleted (i.e. lock released). Try to acquire again
                    }
                }
            }
        }
    }
    catch ( Exception e )
    {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;
        throw e;
    }
    finally
    {
        //清理需要删除节点,如果线程被中段或者线程等待时间超时
        if ( doDelete )
        {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```


``` java
private final Watcher watcher = new Watcher()
{
    @Override
    public void process(WatchedEvent event)
    {
        notifyFromWatcher();
    }
};
```
在`internalLockLoop`方法执行时,未获得锁的线程会wait,直到观察的前置节点发生变动并唤醒.

*以上就是获取锁的源码分析*

#### release Lock
释放锁的代码很简单,需要关注一下重入的情况即可.

``` java
@Override
public void release() throws Exception
{
    /*
        Note on concurrency: a given lockData instance
        can be only acted on by a single thread so locking isn't necessary
     */
    
    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    //并未获得过锁
    if ( lockData == null )
    {
        throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
    }
    //lockCount-1
    int newLockCount = lockData.lockCount.decrementAndGet();
    //如果-1后的结果依然大于0,说明是重入的情况,所以直接返回等待最后一个release
    if ( newLockCount > 0 )
    {
        return;
    }
    //已经释放过锁或者其他的情况
    if ( newLockCount < 0 )
    {
        throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
    }
    try
    {
        //释放锁
        internals.releaseLock(lockData.lockPath);
    }
    finally
    {
        //从threadData中将当前线程remove
        threadData.remove(currentThread);
    }
}
```

``` java
void releaseLock(String lockPath) throws Exception
{
    //设置撤销标识为空
    revocable.set(null);
    //删除锁节点
    deleteOurPath(lockPath);
}
```

>   需要注意acquire与release是成对出现的,否则会出现死锁的情况

### 扩展
`InterProcessMutex`这个类其实是一个基础类,里边提供了很多的扩展,比如它有一个package内可见的构造方法,可以提供一个maxLeases的扩展,后边读写锁的分析就使用到了这个扩展点.
``` java
InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver)
{
    basePath = PathUtils.validatePath(path);
    internals = new LockInternals(client, driver, path, lockName, maxLeases);
}
```


以上就是Curator对于分布式锁的实现,思路比较清晰,代码也比较好懂.