#Apache HttpClient too many close_wait
`HttpClient` `close_wait`
##问题简述
监控系统发现应用服务器大量tcp close_wait。
##问题分析
###close_wait
tcp协议断开链接进行4次挥手，一方率先收到另外一方的FIN时进入CLOSE_WAIT状态。更多内容请参看陈皓的[TCP 的那些事儿](http://coolshell.cn/articles/11564.html)。
使用netstat命令发现close_wait的目的ip是同一个远程主机。针对该主机的访问是程序中的HttpClient发起的。
###HttpClient
`应用程序使用的HttpClient版本为3.1。`在finally块中都调用了HttpMethod的releaseConnection()方法。
###releaseConnection()
HttpMethod的releaseConnection()方法是一个极具迷惑性的方法。调用栈为：
>HttpMethod.releaseConnection()
>>HttpMethodBase.releaseConnection()
>>>HttpMethodBase.ensureConnectionRelease()
>>>>HttpConnection.releaseConnection()
>>>>>HttpConnectionManager.releaseConnection(HttpConnection conn)

看看SimpleHttpConnectionManager的实现，默认是不会close HttpConnection的，而是关闭掉HttpConnection的上一次响应，然后等待复用。
```Java
public void releaseConnection(HttpConnection conn) {
        if (conn != httpConnection) {
            throw new IllegalStateException("Unexpected release of an unknown connection.");
        }
        if (this.alwaysClose) {
            httpConnection.close();
        } else {
            // make sure the connection is reuseable
            finishLastResponse(httpConnection);
        }
        inUse = false;
        // track the time the connection was made idle
        idleStartTime = System.currentTimeMillis();
    }
```
##问题解决
如何正确的关闭连接呢？[Using HttpClient properly to avoid CLOSE_WAIT TCP connections](http://www.tuicool.com/articles/En6niq)提供了三种方法。
####1.设置请求头
```Java
method.setRequestHeader("Connection", "close");
```
####2.调用closeIdleConnections方法
```Java
httpClient.getHttpConnectionManager().closeIdleConnections(0);
```
####3.使用MultiThreadedHttpConnectionManager
##关于HttpClient 4.X.X
参看官方示例[ClientConnectionRelease](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientConnectionRelease.java)

