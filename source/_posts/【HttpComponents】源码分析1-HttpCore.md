---
title: 【HttpComponents】源码分析1-HttpCore
date: 2016-09-21 16:56:21
tags: HttpComponents
categories: Java
---

[HttpComponents](http://hc.apache.org/)前身是大名鼎鼎的[HttpClient](http://hc.apache.org/httpclient-3.x/),考虑到架构或者实现上改动太大,HttpComponents诞生了(重写),它将原有HttpClient进行拆分、模块化.
<!--more-->
这里我选择从底层往上层分析.

1. [HttpCore](http://hc.apache.org/httpcomponents-core-4.4.x/index.html)
2. [HttpClient](http://hc.apache.org/httpcomponents-client-4.5.x/index.html)
3. [HttpAsyncClient](http://hc.apache.org/httpcomponents-asyncclient-4.1.x/index.html)

> Note: 本次源码分析基于HttpComponents 4.x

Let's Go！

### Http协议复习
#### Request
| 请求行(RequestLine) | GET /index.html HTTP/1.1 |
| -----:|-----:|
| 首部(Headers 细分的话为请求、通用、实体三类) | Accept-Encoding: gzip, deflate, sdch |
| CR+LF | 空行 |
| 主体(Body) | xxxx |

例子：
``` 
GET / HTTP/1.1
Host: cn.bing.com
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,zh-TW;q=0.4,ja;q=0.2
```

```
Request       = Request-Line
                       *(( general-header
                        | request-header
                        | entity-header ) CRLF)
                       CRLF
                       [ message-body ]
```

#### Response
| 状态行(StatusLine) | HTTP/1.1 200 OK |
| -----:|-----:|
| 首部(Headers 细分的话为响应、通用、实体三类) | Content-Type: text/html; charset=utf-8 |
| CR+LF | 空行 |
| 主体(Body) | <html>xxxx</html> |

例子：
```
HTTP/1.1 200 OK
Cache-Control: private, max-age=0
Content-Length: 49765
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Vary: Accept-Encoding
Server: Microsoft-IIS/8.5
P3P: CP="NON UNI COM NAV STA LOC CURa DEVa PSAa PSDa OUR IND"
X-MSEdge-Ref: Ref A: 27150F449DE941A79F25F4D540646A24 Ref B: A5C28C846B068A71F2050ADD7537DB8B Ref C: Tue Sep  6 01:37:39 2016 PST
Date: Tue, 06 Sep 2016 08:37:38 GMT

<!DOCTYPE html>.....
```

```
Response      = Status-Line
                      *(( general-header
                       | response-header
                       | entity-header ) CRLF)
                      CRLF
                      [ message-body ]
```
### HttpCore

    请求行: RequestLine->BasicRequestLine

    状态行: StatusLine->BasicStatusLine

    首部:   Header->BasicHeader

    主体:   HttpEntity->AbstractHttpEntity->BasicHttpEntity、StringEntity...

以上是HttpCore对于Http协议进行的抽象以及一些基本的实现.

#### 基本 HttpCore Example (blocking IO)

``` java
public static void main(String[] args) throws Exception {
    HttpProcessor httpproc =
        HttpProcessorBuilder.create().add(new RequestContent()).add(new RequestTargetHost())
            .add(new RequestConnControl()).add(new RequestUserAgent("Test/1.1"))
            .add(new RequestExpectContinue(true)).build();

    HttpRequestExecutor httpexecutor = new HttpRequestExecutor();

    HttpCoreContext coreContext = HttpCoreContext.create();
    HttpHost host = new HttpHost("shuvigoss.win", 80);
    coreContext.setTargetHost(host);

    DefaultBHttpClientConnection conn = new DefaultBHttpClientConnection(8 * 1024);
    ConnectionReuseStrategy connStrategy = DefaultConnectionReuseStrategy.INSTANCE;

    try {
      if (!conn.isOpen()) {
        Socket socket = new Socket(host.getHostName(), host.getPort());
        conn.bind(socket);
      }
      BasicHttpRequest request = new BasicHttpRequest("GET", "/");
      System.out.println(">> Request URI: " + request.getRequestLine().getUri());

      httpexecutor.preProcess(request, httpproc, coreContext);
      HttpResponse response = httpexecutor.execute(request, conn, coreContext);
      httpexecutor.postProcess(response, httpproc, coreContext);

      System.out.println("<< Response: " + response.getStatusLine());
      System.out.println(EntityUtils.toString(response.getEntity()));

      System.out.println("==============");
      if (!connStrategy.keepAlive(response, coreContext)) {
        conn.close();
      } else {
        System.out.println("Connection kept alive...");
      }
    } finally {
      conn.close();
    }
  }
```

上面这个例子就是基于 HttpCore 的请求 Demo , 通过构建请求、建立连接、解析返回、关闭连接等一系列操作实现一次 Http 请求,我会基于这款代码逐步阅读以及分析.

#### HttpProcessor
``` java
HttpProcessor httpproc =
        HttpProcessorBuilder.create().add(new RequestContent()).add(new RequestTargetHost())
            .add(new RequestConnControl()).add(new RequestUserAgent("Test/1.1"))
            .add(new RequestExpectContinue(true)).build();
```

通过`HttpProcessorBuilder`构建一个`HttpProcessor`实例.

``` java
public interface HttpProcessor
    extends HttpRequestInterceptor, HttpResponseInterceptor {

    // no additional methods
}
```
`HttpProcessor` 是用来处理请求前拦截以及请求后拦截,这个理解起来很简单,实际上就是前后拦截处理器.

拿`RequestUserAgent`为例
``` java
public class RequestUserAgent implements HttpRequestInterceptor {

    private final String userAgent;

    public RequestUserAgent(final String userAgent) {
        super();
        this.userAgent = userAgent;
    }

    public RequestUserAgent() {
        this(null);
    }

    @Override
    public void process(final HttpRequest request, final HttpContext context)
        throws HttpException, IOException {
        Args.notNull(request, "HTTP request");
        if (!request.containsHeader(HTTP.USER_AGENT)) {
            String s = null;
            final HttpParams params = request.getParams();
            if (params != null) {
                s = (String) params.getParameter(CoreProtocolPNames.USER_AGENT);
            }
            if (s == null) {
                s = this.userAgent;
            }
            if (s != null) {
                request.addHeader(HTTP.USER_AGENT, s);
            }
        }
    }

}
```
这段代码比较简单,判断请求头是否包含`User-Agent`首部,如果没有,通过request来获取相关内容并且 set 到 Header 上.

#### HttpRequestExecutor
``` java
HttpRequestExecutor httpexecutor = new HttpRequestExecutor();
```
`HttpRequestExecutor` 执行器,负责执行`HttpProcessor`、`HttpConnection` 完成一次 Http 请求
``` java
httpexecutor.preProcess(request, httpproc, coreContext); //执行前拦截
HttpResponse response = httpexecutor.execute(request, conn, coreContext); //请求
httpexecutor.postProcess(response, httpproc, coreContext); //执行后拦截
```

#### HttpContext
``` java
HttpCoreContext coreContext = HttpCoreContext.create();
HttpHost host = new HttpHost("shuvigoss.win", 80);
coreContext.setTargetHost(host);
```
这个是请求上线文的实现,主要包含一些 host, port, param等等内容.

#### HttpConnection
``` java
DefaultBHttpClientConnection conn = new DefaultBHttpClientConnection(8 * 1024);

if (!conn.isOpen()) {
    Socket socket = new Socket(host.getHostName(), host.getPort());
    conn.bind(socket);
}
```
`HttpConnection`是负责具体的连接,由于`HttpCore`并不提供具体的连接实现,所以他做了个连接的抽象(`HttpConnection`),在内部有`socketHolder`、`inbuffer`、`outbuffer`、`requestWriter`、`responseParser`等实现来处理一次Http请求,内部的具体内容有时间再来说.

#### HttpRequestExecutor execute
``` java
public HttpResponse execute(
        final HttpRequest request,
        final HttpClientConnection conn,
        final HttpContext context) throws IOException, HttpException {
    Args.notNull(request, "HTTP request");
    Args.notNull(conn, "Client connection");
    Args.notNull(context, "HTTP context");
    try {
        //通过conn发送request
        HttpResponse response = doSendRequest(request, conn, context);
        //等待返回
        if (response == null) {
            response = doReceiveResponse(request, conn, context);
        }
        return response;
    } catch (final IOException ex) {
        closeConnection(conn);
        throw ex;
    } catch (final HttpException ex) {
        closeConnection(conn);
        throw ex;
    } catch (final RuntimeException ex) {
        closeConnection(conn);
        throw ex;
    }
}

private static void closeConnection(final HttpClientConnection conn) {
    try {
        conn.close();
    } catch (final IOException ignore) {
    }
}
```

``` java
protected HttpResponse doSendRequest(
        final HttpRequest request,
        final HttpClientConnection conn,
        final HttpContext context) throws IOException, HttpException {
    Args.notNull(request, "HTTP request");
    Args.notNull(conn, "Client connection");
    Args.notNull(context, "HTTP context");

    HttpResponse response = null;

    context.setAttribute(HttpCoreContext.HTTP_CONNECTION, conn);
    context.setAttribute(HttpCoreContext.HTTP_REQ_SENT, Boolean.FALSE);
    //写请求头到SessionOutputBuffer->buffer(ByteArrayBuffer)
    conn.sendRequestHeader(request);
    //如果是HttpEntityEnclosingRequest的请求(POST,PUT等),检查请求头是否包含Expect: 100-Continue 头,这个是HTTP/1.1特有的头
    //意思大致是传输大文本时,先询问server是否支持,如果不支持返回417等错误码,如果支持,返回100 continue
    //这时候客户端再讲POST内容传到服务器,如果在等待返回超时时间(3000ms)内服务端没有返回,直接POST,下面代码就是对这个头的处理.
    if (request instanceof HttpEntityEnclosingRequest) {
        // Check for expect-continue handshake. We have to flush the
        // headers and wait for an 100-continue response to handle it.
        // If we get a different response, we must not send the entity.
        boolean sendentity = true;
        final ProtocolVersion ver =
            request.getRequestLine().getProtocolVersion();
        if (((HttpEntityEnclosingRequest) request).expectContinue() &&
            !ver.lessEquals(HttpVersion.HTTP_1_0)) {

            conn.flush();
            // As suggested by RFC 2616 section 8.2.3, we don't wait for a
            // 100-continue response forever. On timeout, send the entity.
            if (conn.isResponseAvailable(this.waitForContinue)) {
                response = conn.receiveResponseHeader();
                if (canResponseHaveBody(request, response)) {
                    conn.receiveResponseEntity(response);
                }
                final int status = response.getStatusLine().getStatusCode();
                if (status < 200) {
                    if (status != HttpStatus.SC_CONTINUE) {
                        throw new ProtocolException(
                                "Unexpected response: " + response.getStatusLine());
                    }
                    // discard 100-continue
                    response = null;
                } else {
                    sendentity = false;
                }
            }
        }
        if (sendentity) {
            conn.sendRequestEntity((HttpEntityEnclosingRequest) request);
        }
    }
    //调用Scoket OutputStream.flush() 真正写入并传输
    conn.flush();
    //记录上下文请求已经发送
    context.setAttribute(HttpCoreContext.HTTP_REQ_SENT, Boolean.TRUE);
    return response;
}
```

``` java
@Override
public void sendRequestHeader(final HttpRequest request)
        throws HttpException, IOException {
    Args.notNull(request, "HTTP request");
    ensureOpen(); //打开socket
    this.requestWriter.write(request); //写入缓冲流SessionOutputBuffer
    onRequestSubmitted(request); //请求提交,子类可根据需求进行定制
    incrementRequestCount(); //记录请求的次数
}
```

``` java
//this.requestWriter.write(request);
@Override
public void write(final T message) throws IOException, HttpException {
    Args.notNull(message, "HTTP message");
    //写RequestLine GET / HTTP/1.1
    writeHeadLine(message); 
    //写Header 
    for (final HeaderIterator it = message.headerIterator(); it.hasNext(); ) {
        final Header header = it.nextHeader();
        this.sessionBuffer.writeLine
            (lineFormatter.formatHeader(this.lineBuf, header));
    }
    //清理lineBuf
    this.lineBuf.clear();
    //写空行
    this.sessionBuffer.writeLine(this.lineBuf);
}
```
这里就是完整的请求行+首部的写入,里边的 LineBuffer(CharArrayBuffer)就是每行的buffer,可重用,每次洗完都要清理.sessionBuffer是一个连接最终需要传输的报文buffer.

接下来就是处理返回了,由于 Socket 是阻塞的,调用 OutputStream.write 后,服务端就会往InputStream写响应内容.
``` java
protected HttpResponse doReceiveResponse(
        final HttpRequest request,
        final HttpClientConnection conn,
        final HttpContext context) throws HttpException, IOException {
    Args.notNull(request, "HTTP request");
    Args.notNull(conn, "Client connection");
    Args.notNull(context, "HTTP context");
    HttpResponse response = null;
    int statusCode = 0;

    while (response == null || statusCode < HttpStatus.SC_OK) {
        //创建response并且将头信息读入Response
        response = conn.receiveResponseHeader();
        //读取body
        if (canResponseHaveBody(request, response)) {
            conn.receiveResponseEntity(response);
        }
        statusCode = response.getStatusLine().getStatusCode();

    } // while intermediate response

    return response;
}
```

``` java
 @Override
public HttpResponse receiveResponseHeader() throws HttpException, IOException {
    ensureOpen();
    final HttpResponse response = this.responseParser.parse();
    onResponseReceived(response);
    if (response.getStatusLine().getStatusCode() >= HttpStatus.SC_OK) {
        incrementResponseCount();
    }
    return response;
}
```

由于解析的过程比较复杂,这里就不具体写了,有个关键点需要记住,由于Socket InputStream 是不能重复读取的,所以在整个解析过程会先读取StatusLine,用于创建Response,而body的数据会封装成Entity(InputStream)用于客户端消费.

> 基于传统的阻塞IO就介绍完了,通过这个阻塞IO的学习能够了解到HttpCore内部基本的一些功能以及组件.