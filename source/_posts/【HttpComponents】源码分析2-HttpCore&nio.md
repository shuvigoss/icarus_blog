---
title: 【HttpComponents】源码分析2-HttpCore&nio
date: 2016-09-23 17:00:33
tags: HttpComponents
categories: Java
---

前边介绍了HttpCore里基于传统阻塞IO实现,接下来这篇会比较长,主要是在架构层面上介绍HttpCore+HttpCore NIO.

NIO是什么我这里就不具体介绍了,如果有兴趣可以去看别人写的文档.下面这两个链接介绍内容都是一致的,基于老外写的一篇文章.

<!--more-->
- [https://java-nio.avenwu.net/index.html](https://java-nio.avenwu.net/index.html)
- [http://ifeve.com/overview/](http://ifeve.com/overview/)


### Reactor模式
Reactor是处理高并发所设计的一种模式,NIO完全诠释了Reactor设计模式,如果你熟悉了Java的NIO,那么对于Reactor模式也算是有所了解了.我找了很多文章,介绍的都不是很具体,结合NIO和HttpCore NIO来理解Reactor是个不错的选择.

> 这里有一些文档地址,强烈推荐看一下Doug Lea的文章,里边的多Reactor模式就是HttpCore NIO所使用的。

- [http://www.blogs8.cn/posts/A1h57fe](http://www.blogs8.cn/posts/A1h57fe)
- [https://travisliu.gitbooks.io/learn-eventmachine/content/reactorpattern.html](https://travisliu.gitbooks.io/learn-eventmachine/content/reactorpattern.html)
- [http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)


![](/images/reactor.png)

多Reactor设计的好处在于mainReactor负责接受请求而具体的事情由运行在线程池中的subReactor处理的(read、write).HttpCore NIO与这个设计很像,而且也是基于Doug Lea的Reactor所设计的.他在官方文档里也说了.

### 基本 HttpCore NIO Example (non-blocking IO)

``` java
HttpProcessor httpproc =
        HttpProcessorBuilder.create()
            // Use standard client-side protocol interceptors
            .add(new RequestContent()).add(new RequestTargetHost()).add(new RequestConnControl())
            .add(new RequestUserAgent("Test/1.1")).add(new RequestExpectContinue(true)).build();
// Create client-side HTTP protocol handler
HttpAsyncRequestExecutor protocolHandler = new HttpAsyncRequestExecutor();
// Create client-side I/O event dispatch
final IOEventDispatch ioEventDispatch =
    new DefaultHttpClientIODispatch(protocolHandler, ConnectionConfig.DEFAULT);
// Create client-side I/O reactor
final ConnectingIOReactor ioReactor = new DefaultConnectingIOReactor();
// Create HTTP connection pool
BasicNIOConnPool pool = new BasicNIOConnPool(ioReactor, ConnectionConfig.DEFAULT);
// Limit total number of connections to just two
pool.setDefaultMaxPerRoute(2);
pool.setMaxTotal(2);
// Run the I/O reactor in a separate thread
Thread t = new Thread(new Runnable() {

  public void run() {
    try {
      // Ready to go!
      ioReactor.execute(ioEventDispatch);
    } catch (InterruptedIOException ex) {
      System.err.println("Interrupted");
    } catch (IOException e) {
      System.err.println("I/O error: " + e.getMessage());
    }
    System.out.println("Shutdown");
  }

});
// Start the client thread
t.start();
// Create HTTP requester
HttpAsyncRequester requester = new HttpAsyncRequester(httpproc);
// Execute HTTP GETs to the following hosts and
HttpHost[] targets =
    new HttpHost[] {new HttpHost("shuvigoss.win", 80, "http"),
        new HttpHost("shuvigoss.win", 80, "http"), new HttpHost("shuvigoss.win", 80, "http")};
final CountDownLatch latch = new CountDownLatch(targets.length);
for (final HttpHost target : targets) {
  BasicHttpRequest request = new BasicHttpRequest("GET", "/");
  HttpCoreContext coreContext = HttpCoreContext.create();
  requester.execute(new BasicAsyncRequestProducer(target, request),
      new BasicAsyncResponseConsumer(), pool, coreContext,
      // Handle HTTP response from a callback
      new FutureCallback<HttpResponse>() {

        public void completed(final HttpResponse response) {
          latch.countDown();
          System.out.println(target + "->" + response.getStatusLine());
        }

        public void failed(final Exception ex) {
          latch.countDown();
          System.out.println(target + "->" + ex);
        }

        public void cancelled() {
          latch.countDown();
          System.out.println(target + " cancelled");
        }

      });
}
latch.await();
System.out.println("Shutting down I/O reactor");
ioReactor.shutdown();
System.out.println("Done");
```

![](/images/nio-reactor.png)

上图是对NIO Reactor各角色的划分,其中MainReactor运行于线程1,其他SubReactor根据CPU数量运行于其他线程.其中MainReactor主要负责处理`SelectionKey.OP_CONNECT`.SubReactor负责`SelectionKey.OP_READ`、`SelectionKey.OP_WRITE`.

#### MainReactor IOReactor execute(IOEventDispatch eventDispatch)
``` java
Thread t = new Thread(new Runnable() {

  public void run() {
    try {
      // Ready to go!
      ioReactor.execute(ioEventDispatch);
    } catch (InterruptedIOException ex) {
      System.err.println("Interrupted");
    } catch (IOException e) {
      System.err.println("I/O error: " + e.getMessage());
    }
    System.out.println("Shutdown");
  }

});
// Start the client thread
t.start();
```
这里就已经启动了MainReactor Loop.

``` java
public void execute(
        final IOEventDispatch eventDispatch) throws InterruptedIOException, IOReactorException {
    Args.notNull(eventDispatch, "Event dispatcher");
    synchronized (this.statusLock) {
        if (this.status.compareTo(IOReactorStatus.SHUTDOWN_REQUEST) >= 0) {
            this.status = IOReactorStatus.SHUT_DOWN;
            this.statusLock.notifyAll();
            return;
        }
        Asserts.check(this.status.compareTo(IOReactorStatus.INACTIVE) == 0,
                "Illegal state %s", this.status);
        this.status = IOReactorStatus.ACTIVE;
        // Start I/O dispatchers
        // 创建SubReactor,数量默认为CPU数量,并且运行在Worker线程上
        for (int i = 0; i < this.dispatchers.length; i++) {
            final BaseIOReactor dispatcher = new BaseIOReactor(this.selectTimeout, this.interestOpsQueueing);
            dispatcher.setExceptionHandler(exceptionHandler);
            this.dispatchers[i] = dispatcher;
        }
        for (int i = 0; i < this.workerCount; i++) {
            final BaseIOReactor dispatcher = this.dispatchers[i];
            this.workers[i] = new Worker(dispatcher, eventDispatch);
            this.threads[i] = this.threadFactory.newThread(this.workers[i]);
        }
    }
    try {

        for (int i = 0; i < this.workerCount; i++) {
            if (this.status != IOReactorStatus.ACTIVE) {
                return;
            }
            this.threads[i].start();
        }

        for (;;) {
            final int readyCount;
            try {
                //获取连接Key事件 SelectionKey.OP_CONNECT
                readyCount = this.selector.select(this.selectTimeout);
            } catch (final InterruptedIOException ex) {
                throw ex;
            } catch (final IOException ex) {
                throw new IOReactorException("Unexpected selector failure", ex);
            }

            if (this.status.compareTo(IOReactorStatus.ACTIVE) == 0) {
                //处理连接
                processEvents(readyCount);
            }

            // Verify I/O dispatchers
            for (int i = 0; i < this.workerCount; i++) {
                final Worker worker = this.workers[i];
                final Exception ex = worker.getException();
                if (ex != null) {
                    throw new IOReactorException(
                            "I/O dispatch worker terminated abnormally", ex);
                }
            }

            if (this.status.compareTo(IOReactorStatus.ACTIVE) > 0) {
                break;
            }
        }

    } catch (final ClosedSelectorException ex) {
        addExceptionEvent(ex);
    } catch (final IOReactorException ex) {
        if (ex.getCause() != null) {
            addExceptionEvent(ex.getCause());
        }
        throw ex;
    } finally {
        doShutdown();
        synchronized (this.statusLock) {
            this.status = IOReactorStatus.SHUT_DOWN;
            this.statusLock.notifyAll();
        }
    }
}
```
上面的代码其实逻辑很简单.
1. 创建SubReactor(Worker Thread)并且运行.
2. 监听Connect事件,并且处理

这里有个疑问,并没有给这个selector注册过OP_CONNECT,这块比较巧妙,后边会介绍.

#### HttpAsyncRequester execute
下面是创建Request并且由`HttpAsyncRequester`执行
``` java
for (final HttpHost target : targets) {
  BasicHttpRequest request = new BasicHttpRequest("GET", "/");
  HttpCoreContext coreContext = HttpCoreContext.create();
  requester.execute(new BasicAsyncRequestProducer(target, request),
      new BasicAsyncResponseConsumer(), pool, coreContext,
      // Handle HTTP response from a callback
      new FutureCallback<HttpResponse>() {

        public void completed(final HttpResponse response) {
          latch.countDown();
          System.out.println(target + "->" + response.getStatusLine());
        }

        public void failed(final Exception ex) {
          latch.countDown();
          System.out.println(target + "->" + ex);
        }

        public void cancelled() {
          latch.countDown();
          System.out.println(target + " cancelled");
        }

      });
}
```

通过连接池,创建Request.
``` java
public Future<E> lease(
            final T route, final Object state,
            final long connectTimeout, final long leaseTimeout, final TimeUnit tunit,
            final FutureCallback<E> callback) {
        Args.notNull(route, "Route");
        Args.notNull(tunit, "Time unit");
        Asserts.check(!this.isShutDown.get(), "Connection pool shut down");
        final BasicFuture<E> future = new BasicFuture<E>(callback);
        this.lock.lock();
        try {
            final long timeout = connectTimeout > 0 ? tunit.toMillis(connectTimeout) : 0;
            final LeaseRequest<T, C, E> request = new LeaseRequest<T, C, E>(route, state, timeout, leaseTimeout, future);
            final boolean completed = processPendingRequest(request);
            if (!request.isDone() && !completed) {
                this.leasingRequests.add(request);
            }
            if (request.isDone()) {
                this.completedRequests.add(request);
            }
        } finally {
            this.lock.unlock();
        }
        fireCallbacks();
        return future;
    }

```
这里边的`processPendingRequest(request)`很重要,里边代码量有点大,我把主要的核心代码贴出来分析.

``` java
// New connection is needed
final int maxPerRoute = getMax(route);
// Shrink the pool prior to allocating a new connection
final int excess = Math.max(0, pool.getAllocatedCount() + 1 - maxPerRoute);
if (excess > 0) {
    for (int i = 0; i < excess; i++) {
        final E lastUsed = pool.getLastUsed();
        if (lastUsed == null) {
            break;
        }
        lastUsed.close();
        this.available.remove(lastUsed);
        pool.remove(lastUsed);
    }
}

if (pool.getAllocatedCount() < maxPerRoute) {
    final int totalUsed = this.pending.size() + this.leased.size();
    final int freeCapacity = Math.max(this.maxTotal - totalUsed, 0);
    if (freeCapacity == 0) {
        return false;
    }
    final int totalAvailable = this.available.size();
    if (totalAvailable > freeCapacity - 1) {
        if (!this.available.isEmpty()) {
            final E lastUsed = this.available.removeLast();
            lastUsed.close();
            final RouteSpecificPool<T, C, E> otherpool = getPool(lastUsed.getRoute());
            otherpool.remove(lastUsed);
        }
    }

    final SocketAddress localAddress;
    final SocketAddress remoteAddress;
    try {
        remoteAddress = this.addressResolver.resolveRemoteAddress(route);
        localAddress = this.addressResolver.resolveLocalAddress(route);
    } catch (final IOException ex) {
        request.failed(ex);
        return false;
    }
    //调用ioreactor(DefaultConnectingIOReactor) 进行connect
    final SessionRequest sessionRequest = this.ioreactor.connect(
            remoteAddress, localAddress, route, this.sessionRequestCallback);
    final int timout = request.getConnectTimeout() < Integer.MAX_VALUE ?
            (int) request.getConnectTimeout() : Integer.MAX_VALUE;
    sessionRequest.setConnectTimeout(timout);
    this.pending.add(sessionRequest);
    pool.addPending(sessionRequest, request.getFuture());
    return true;
} else {
    return false;
}
```
这段代码很重要`this.ioreactor.connect`,实际上调用的是`DefaultConnectingIOReactor.connect`方法.

接下来就是注册CONNET事件的核心代码了.
``` java
@Override
public SessionRequest connect(
        final SocketAddress remoteAddress,
        final SocketAddress localAddress,
        final Object attachment,
        final SessionRequestCallback callback) {
    Asserts.check(this.status.compareTo(IOReactorStatus.ACTIVE) <= 0,
        "I/O reactor has been shut down");
    final SessionRequestImpl sessionRequest = new SessionRequestImpl(
            remoteAddress, localAddress, attachment, callback);
    sessionRequest.setConnectTimeout(this.config.getConnectTimeout());

    this.requestQueue.add(sessionRequest);
    this.selector.wakeup();

    return sessionRequest;
}
```
创建一个SessionRequest并放入`DefaultConnectingIOReactor.requestQueue`内.

``` java
private void processSessionRequests() throws IOReactorException {
    SessionRequestImpl request;
    while ((request = this.requestQueue.poll()) != null) {
        if (request.isCompleted()) {
            continue;
        }
        final SocketChannel socketChannel;
        try {
            socketChannel = SocketChannel.open();
        } catch (final IOException ex) {
            request.failed(ex);
            return;
        }
        try {
            validateAddress(request.getLocalAddress());
            validateAddress(request.getRemoteAddress());

            socketChannel.configureBlocking(false);
            prepareSocket(socketChannel.socket());

            if (request.getLocalAddress() != null) {
                final Socket sock = socketChannel.socket();
                sock.setReuseAddress(this.config.isSoReuseAddress());
                sock.bind(request.getLocalAddress());
            }
            final boolean connected = socketChannel.connect(request.getRemoteAddress());
            if (connected) {
                final ChannelEntry entry = new ChannelEntry(socketChannel, request);
                addChannel(entry);
                continue;
            }
        } catch (final IOException ex) {
            closeChannel(socketChannel);
            request.failed(ex);
            return;
        }

        final SessionRequestHandle requestHandle = new SessionRequestHandle(request);
        try {
            final SelectionKey key = socketChannel.register(this.selector, SelectionKey.OP_CONNECT,
                    requestHandle);
            request.setKey(key);
        } catch (final IOException ex) {
            closeChannel(socketChannel);
            throw new IOReactorException("Failure registering channel " +
                    "with the selector", ex);
        }
    }
}
```
`processSessionRequests`循环获取requestQueue,如果有元素将会创建一个Channel并且注册`SelectionKey.OP_CONNECT`到selector

最后,处理连接finishConnect,将新的Channel加到SubReactor内.
``` java
private void processEvent(final SelectionKey key) {
    try {

        if (key.isConnectable()) {

            final SocketChannel channel = (SocketChannel) key.channel();
            // Get request handle
            final SessionRequestHandle requestHandle = (SessionRequestHandle) key.attachment();
            final SessionRequestImpl sessionRequest = requestHandle.getSessionRequest();

            // Finish connection process
            try {
                channel.finishConnect();
            } catch (final IOException ex) {
                sessionRequest.failed(ex);
            }
            key.cancel();
            key.attach(null);
            if (!sessionRequest.isCompleted()) {
                addChannel(new ChannelEntry(channel, sessionRequest));
            } else {
                try {
                    channel.close();
                } catch (IOException ignore) {
                }
            }
        }

    } catch (final CancelledKeyException ex) {
        final SessionRequestHandle requestHandle = (SessionRequestHandle) key.attachment();
        key.attach(null);
        if (requestHandle != null) {
            final SessionRequestImpl sessionRequest = requestHandle.getSessionRequest();
            if (sessionRequest != null) {
                sessionRequest.cancel();
            }
        }
    }
}
```
尽量平均的分配到SubReactor
``` java
protected void addChannel(final ChannelEntry entry) {
    // Distribute new channels among the workers
    final int i = Math.abs(this.currentWorker++ % this.workerCount);
    this.dispatchers[i].addChannel(entry);
}
```

到此连接创建的核心代码就读完了,稍微有那么一点点的绕.

#### SubReactor IOReactor execute(IOEventDispatch eventDispatch)
这里的SubReactor具体实现为`BaseIOReactor`
``` java
@Override
public void execute(
        final IOEventDispatch eventDispatch) throws InterruptedIOException, IOReactorException {
    Args.notNull(eventDispatch, "Event dispatcher");
    this.eventDispatch = eventDispatch;
    execute();
}   
```
``` java
protected void execute() throws InterruptedIOException, IOReactorException {
    this.status = IOReactorStatus.ACTIVE;

    try {
        for (;;) {

            final int readyCount;
            try {
                readyCount = this.selector.select(this.selectTimeout);
            } catch (final InterruptedIOException ex) {
                throw ex;
            } catch (final IOException ex) {
                throw new IOReactorException("Unexpected selector failure", ex);
            }

            if (this.status == IOReactorStatus.SHUT_DOWN) {
                // Hard shut down. Exit select loop immediately
                break;
            }

            if (this.status == IOReactorStatus.SHUTTING_DOWN) {
                // Graceful shutdown in process
                // Try to close things out nicely
                closeSessions();
                closeNewChannels();
            }

            // Process selected I/O events
            if (readyCount > 0) {
                processEvents(this.selector.selectedKeys());
            }

            // Validate active channels
            validate(this.selector.keys());

            // Process closed sessions
            processClosedSessions();

            // If active process new channels
            if (this.status == IOReactorStatus.ACTIVE) {
                processNewChannels();
            }

            // Exit select loop if graceful shutdown has been completed
            if (this.status.compareTo(IOReactorStatus.ACTIVE) > 0
                    && this.sessions.isEmpty()) {
                break;
            }

            if (this.interestOpsQueueing) {
                // process all pending interestOps() operations
                processPendingInterestOps();
            }

        }

    } catch (final ClosedSelectorException ignore) {
    } finally {
        hardShutdown();
        synchronized (this.statusMutex) {
            this.statusMutex.notifyAll();
        }
    }
}
```
SubReactor具体处理2个比较重要的事情
1. `processEvents(this.selector.selectedKeys());` 处理具体读写事件
2. `processNewChannels();` 处理新的Channel

``` java
private void processNewChannels() throws IOReactorException {
    ChannelEntry entry;
    while ((entry = this.newChannels.poll()) != null) {

        final SocketChannel channel;
        final SelectionKey key;
        try {
            channel = entry.getChannel();
            channel.configureBlocking(false);
            key = channel.register(this.selector, SelectionKey.OP_READ);
        } catch (final ClosedChannelException ex) {
            final SessionRequestImpl sessionRequest = entry.getSessionRequest();
            if (sessionRequest != null) {
                sessionRequest.failed(ex);
            }
            return;

        } catch (final IOException ex) {
            throw new IOReactorException("Failure registering channel " +
                    "with the selector", ex);
        }

        final SessionClosedCallback sessionClosedCallback = new SessionClosedCallback() {

            @Override
            public void sessionClosed(final IOSession session) {
                queueClosedSession(session);
            }

        };

        InterestOpsCallback interestOpsCallback = null;
        if (this.interestOpsQueueing) {
            interestOpsCallback = new InterestOpsCallback() {

                @Override
                public void addInterestOps(final InterestOpEntry entry) {
                    queueInterestOps(entry);
                }

            };
        }

        final IOSession session;
        try {
            session = new IOSessionImpl(key, interestOpsCallback, sessionClosedCallback);
            int timeout = 0;
            try {
                timeout = channel.socket().getSoTimeout();
            } catch (final IOException ex) {
                // Very unlikely to happen and is not fatal
                // as the protocol layer is expected to overwrite
                // this value anyways
            }

            session.setAttribute(IOSession.ATTACHMENT_KEY, entry.getAttachment());
            session.setSocketTimeout(timeout);
        } catch (final CancelledKeyException ex) {
            continue;
        }
        try {
            this.sessions.add(session);
            final SessionRequestImpl sessionRequest = entry.getSessionRequest();
            if (sessionRequest != null) {
                sessionRequest.completed(session);
            }
            key.attach(session);
            sessionCreated(key, session);
        } catch (final CancelledKeyException ex) {
            queueClosedSession(session);
            key.attach(null);
        }
    }
}
```
处理新的Channel逻辑也比较简单
1. 首先注册`SelectionKey.OP_READ`
2. 然后创建`IOSession`attach到Key上,从`IOSession`接口上看到,它主要负责对于事件的派发
3. `sessionCreated(key, session)`会调用`DefaultHttpClientIODispatch.onConnected`方法.这个分发器会调用`HttpAsyncRequestExecutor`(handler)来做具体连接后的事情.
4. `HttpAsyncRequestExecutor`就是真正用于处理请求发送以及返回解析的处理类.


``` java
@Override
protected void sessionCreated(final SelectionKey key, final IOSession session) {
    try {
        this.eventDispatch.connected(session);
    } catch (final CancelledKeyException ex) {
        queueClosedSession(session);
    } catch (final RuntimeException ex) {
        handleRuntimeException(ex);
    }
}
```
``` java
@Override
public void connected(final IOSession session) {
    @SuppressWarnings("unchecked")
    T conn = (T) session.getAttribute(IOEventDispatch.CONNECTION_KEY);
    try {
        if (conn == null) {
            conn = createConnection(session);
            session.setAttribute(IOEventDispatch.CONNECTION_KEY, conn);
        }
        onConnected(conn);
        final SSLIOSession ssliosession = (SSLIOSession) session.getAttribute(
                SSLIOSession.SESSION_KEY);
        if (ssliosession != null) {
            try {
                synchronized (ssliosession) {
                    if (!ssliosession.isInitialized()) {
                        ssliosession.initialize();
                    }
                }
            } catch (final IOException ex) {
                onException(conn, ex);
                ssliosession.shutdown();
            }
        }
    } catch (final RuntimeException ex) {
        session.shutdown();
        throw ex;
    }
}
```
``` java
@Override
protected void onConnected(final DefaultNHttpClientConnection conn) {
    final Object attachment = conn.getContext().getAttribute(IOSession.ATTACHMENT_KEY);
    try {
        this.handler.connected(conn, attachment);
    } catch (final Exception ex) {
        this.handler.exception(conn, ex);
    }
}
```
``` java
@Override
public void connected(
        final NHttpClientConnection conn,
        final Object attachment) throws IOException, HttpException {
    final State state = new State();
    final HttpContext context = conn.getContext();
    context.setAttribute(HTTP_EXCHANGE_STATE, state);
    requestReady(conn);
}
```
``` java
@Override
public void requestReady(
        final NHttpClientConnection conn) throws IOException, HttpException {
    final State state = getState(conn);
    Asserts.notNull(state, "Connection state");
    Asserts.check(state.getRequestState() == MessageState.READY ||
                    state.getRequestState() == MessageState.COMPLETED,
            "Unexpected request state %s", state.getRequestState());

    if (state.getRequestState() == MessageState.COMPLETED) {
        conn.suspendOutput();
        return;
    }
    final HttpContext context = conn.getContext();
    final HttpAsyncClientExchangeHandler handler;
    synchronized (context) {
        handler = getHandler(conn);
        if (handler == null || handler.isDone()) {
            conn.suspendOutput();
            return;
        }
    }
    final boolean pipelined = handler.getClass().getAnnotation(Pipelined.class) != null;

    final HttpRequest request = handler.generateRequest();
    if (request == null) {
        conn.suspendOutput();
        return;
    }
    final ProtocolVersion version = request.getRequestLine().getProtocolVersion();
    if (pipelined && version.lessEquals(HttpVersion.HTTP_1_0)) {
        throw new ProtocolException(version + " cannot be used with request pipelining");
    }
    state.setRequest(request);
    if (pipelined) {
        state.getRequestQueue().add(request);
    }
    if (request instanceof HttpEntityEnclosingRequest) {
        final boolean expectContinue = ((HttpEntityEnclosingRequest) request).expectContinue();
        if (expectContinue && pipelined) {
            throw new ProtocolException("Expect-continue handshake cannot be used with request pipelining");
        }
        conn.submitRequest(request);
        if (expectContinue) {
            final int timeout = conn.getSocketTimeout();
            state.setTimeout(timeout);
            conn.setSocketTimeout(this.waitForContinue);
            state.setRequestState(MessageState.ACK_EXPECTED);
        } else {
            final HttpEntity entity = ((HttpEntityEnclosingRequest) request).getEntity();
            if (entity != null) {
                state.setRequestState(MessageState.BODY_STREAM);
            } else {
                handler.requestCompleted();
                state.setRequestState(pipelined ? MessageState.READY : MessageState.COMPLETED);
            }
        }
    } else {
        conn.submitRequest(request);//会通过IOSession注册OP_WRITE事件
        handler.requestCompleted();
        state.setRequestState(pipelined ? MessageState.READY : MessageState.COMPLETED);
    }
}
```
这里`conn.submitRequest(request);`会注册OP_WRITE到selector.

#### ending
这里框架上的东西差不多就完了,总结一下.

* `DefaultConnectingIOReactor` 负责处理所有连接,创建Channel并将Channel分配到SubReactor(BaseIOReactor)
* `HttpAsyncRequestExecutor`作用是Handler,处理所有连接创建、读、写等等
* `DefaultHttpClientIODispatch` 主要用于派发具体的事件
* `BaseIOReactor`负责读写事件的处理

> 其实还有很多细节实在没法写了,具体还是从代码层面看！
