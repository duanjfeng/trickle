#TLS: Protocol details and troubleshooting
#TLS：协议详情和故障诊断
`TLS` 

##引子
项目组使用TLS的场景越来越多，为了提高通信安全，所有开放到互联网的数据几乎都要求通过TLS协议访问。<br/>
使用TLS的场景也越来越复杂，不像一般的网站，通常是处理浏览器发起的https请求（而且这种请求通常是单向认证的，即客户端认证服务器），我们在使用时还要处理HttpClient模拟浏览器的请求、CDN节点请求、使用硬件安全模块中的证书、双向认证等复杂场景。<br/>
本文简要介绍一下TLS协议（握手过程），结合日常使用时碰到的问题尝试提供一些故障诊断的思路。<br/>

##什么是TLS
###TLS的含义
TLS（Transport Layer Security），前身为SSL（Secure Socket Layer），用于保障网络通信安全的安全协议。<br/>
安全协议是指采用密码学方法提供安全相关特性的协议。除了TLS之外，还有IPsec、Diffie-Hellman等安全协议。<br/>
###网络层次
TLS工作在TCP/IP协议栈的应用层，位于传输层之上，从而保障应用层数据的安全性。<br/>
相比而言，IPsec工作于TCP/IP协议栈的网络层，工作于传输模式时保障IP报文体的安全性，工作于隧道模式时保障整个IP报文的安全性。<br/>
###应用场景
TLS采用非对称密钥算法协商出会话密钥（对称密钥），应用数据被对称密钥加密后传输。<br/>
由于采用了非对称密码技术、对称密码技术、安全哈希技术，从理论上TLS可以实现通信双方身份认证和抗抵赖、数据加密、数据完整性和防篡改。<br/>
最常见的应用场景是HTTPS协议。相比TCP协议明文传输HTTP协议内容，在HTTPS协议中使用了TLS协议机制完成了HTTP内容的加解密。从而提高了HTTP数据访问的安全性。<br/>
其他一些协议也会使用TLS以提高通信的安全性，例如FTPS协议,xmpp协议、mqtt协议等。<br/>
###发展历史。
TLS的前身为SSL，由Netscape（网景）公司开发，目的是提高HTTP协议的安全性。先后开发了SSL 1.0（未公开）、2.0（1995）、3.0（1996）三个版本。SSL 3.0至今依然在使用。<br/>
TLS 1.0发布于1999年，是对SSL 3.0的升级。随后发布的TLS 1.1（2006）、TLS 1.2（2008）均是对其前一个版本的优化升级。TLS 1.3的草案已经于2015年发布。<br/>
SSL协议存在安全隐患，因此越来越多的主流平台、网站和浏览器不建议使用。TLS协议也是建议使用当前最高版本TLS 1.2。<br/>
以Oracle JDK为例，JDK 6支持SSL 3.0和TLS 1.0（默认协议），JDK 7支持SSL 3.0至TLS 1.2，以TLS 1.0为默认协议，JDK 8支持SSL 3.0至TLS 1.2，以TLS 1.2为默认协议。<br/>
而Google Chrome浏览器和Mozilla Firefox浏览器的最新版本已经不再支持SSL系列协议。<br/>

##TLS握手过程

					Client								Server
	1. Client hello						--->		
										<---	2. Server hello
										<---	3. Certificate (optional)
										<---	4. Certificate request (optional)
										<---	5. Server key exchange (optional)
										<---	6. Server hello done
	7. Certificate (optional)			--->
	8. Client key exchange				--->
	9. Certificate verify (optional)	--->
	10. Change cipher spec				--->
	11. Finished						--->
										<---	12. Change cipher spec
										<---	13. Finished
	14. Encrypted data					<-->	14. Encrypted data
	15. Close messages					<-->	15. Close messages

1. Client hello<br/>
客户端发送自己所支持的TLS最高版本号和密码套件。密码套件信息包括密码算法和密钥长度。<br/>
2. Server hello<br/>
服务器选择双方均支持的TLS协议的最高版本和最佳密码套件，发送给客户端。<br/>
3. Certificate<br/>
服务器发送证书或者证书链。这个消息是可选的，在服务器端认证时需要。通常，TLS协议是要求验证服务器的，例如https场景。<br/>
4. Certificate request<br/>
如果需要认证客户端，则服务器向客户端发送一个证书请求。在互联网环境中通常只认证服务器，很少认证客户端。少数场景是双向认证的。<br/>
5. Server key exchange<br/>
如果消息3中证书信息不足以完成密钥交换，服务器会发送一个密钥交换信息。例如，当使用Diffie-Hellman算法时，通过这个消息发送一个DH公钥。<br/>
6. Server hello done<br/>
服务器通知客户端自己已完成初始协商。<br/>
7. Certificate<br/>
如果服务器向客户端发送了消息4，那么客户端需要通过本消息发送证书或者证书链。<br/>
8. Client key exchange<br/>
客户端发送生成对称加密密钥的相关信息。TLS使用RSA算法时，客户端会使用服务器公钥将信息加密发送；TLS使用Diffie-Hellman算法时，客户端会发送自己的DH公钥。<br/>
9. Certificate verify<br/>
如果客户端发送了消息7，那么客户端发送本消息，使得服务器可以认证自己的身份。客户端对消息进行数字签名，服务器验证数字签名。<br/>
10. Change cipher spec<br/>
客户端通知服务器切换到加密模式。<br/>
11. Finished<br/>
客户端通知服务器已经完成协商过程，准备好进行安全数据通信。<br/>
12. Change cipher spec<br/>
服务器通知客户端切换到加密模式。<br/>
13. Finished<br/>
服务器通知客户端已经完成协商过程，准备好进行安全数据通信。这一步是TLS握手的最后一步。<br/>
14. Encrypted data<br/>
服务器和客户端使用协商好的对称密钥对数据进行加密，传输加密数据。这时服务器和客户端也可以重新握手协商。<br/>
15. Close Messages<br/>
在连接结束时，双方发送关闭消息。<br/>

##TLS配置
服务器需要使用TLS协议提供服务时需要进行TLS相关的配置。<br/>
根据TLS协议，通常需要配置服务器的公私钥，是否需要验证客户端，如果需要验证客户端还需要配置信任库。<br/>
下面以Tomcat 7配置https访问为例来简要说明一下，以下xml是Tomcat 7的默认配置。<br/>
<br/>
```xml
	<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
		maxThreads="150" scheme="https" secure="true"
		clientAuth="false" sslProtocol="TLS" />
```
<br/>
<br/>
在Connector中 设置SSLEnabled为true，打开Connector对TLS的支持。同时需要设置schema为https，secure为true，使得Servlet能够获取正确的信息，具体为request.getScheme()和request.isSecure()方法。<br/>
在SSLEnabled开关打开后，开始TLS协议相关的各项配置如下。<br/>
keyAlias：在keystore中服务器私钥和证书的entry的alias。不设置时默认读取第一个。<br/>
keyPass：上述entry的访问口令。默认值是changeit。<br/>
keystoreFile：包含服务器私钥和证书的keystore文件。<br/>
keystorePass：上述keystore文件的访问口令。默认使用keyPass。<br/>
keystoreType：上述keystore的类型，默认为JKS。<br/>
sslProtocol：ssl协议，默认为TLS。<br/>
clientAuth：是否验证客户端。如果为true，则配置下面的参数。<br/>
truststoreFile：信任库的keystore，默认为系统属性javax.net.ssl.trustStore的值。<br/>
truststorePass：上述信任库的访问口令，默认为系统属性javax.net.ssl.trustStorePassword的值。<br/>
truststoreType：上述信任库的类型，默认为系统属性javax.net.ssl.trustStoreType的值。<br/>

##TLS故障诊断
###客户端访问错误提示
访问https服务时，如果出现故障，浏览器、cURL等客户端通常会有错误提示。例如，Chrome提示“ERR_CERT_DATE_INVALID”意味着服务器证书不在有效期。<br/>
这个方法适合客户端查错。<br/>
###握手日志
可以将握手过程的日志打印出来，看看是在握手的哪一步出现问题。<br/>
以Java为例，如果想使虚拟机输出TLS握手日志，需要在启动参数增加属性：-Djavax.net.debug=ssl:handshake:verbose。<br/>
关于日志配置更多内容，请参看JDK文档，[Java Secure Socket Extension (JSSE) Reference Guide](http://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html\#Debug)。
###网络抓包
网络数据包分析工具wireshark可以解析TLS包。<br/>
过滤器类似：ssl.record.version == 0x0301 or ssl.record.version == 0x0303。0x0301为TLS 1.0协议，0x0302为TLS 1.1协议，0X0303为TLS 1.2协议，以此类推。<br/>

##TLS常见问题
###单向认证与双向认证
大部分场景为单向认证，因此客户端不需要提供证书，服务器也不需要配置信任库。<br/>
极少数情况下认证是双向的。例如支付宝会给用户颁发数据证书，以提高pc浏览器支付的便捷性。
###服务器域名与证书DN
为了防止冒名顶替，一般客户端要求服务器域名与服务器证书DN中CommonName部分保持一致。例如，www.alipay.com发给浏览器的证书的CommonName必须为www.alipay.com，百度发给浏览器的证书的CommonName为*.baidu.com。<br/>
如果不一致，大部分浏览器的最新版本会拒绝信任服务器证书，因而无法完成https握手。<br/>
###证书密钥用法
证书密钥用法是证书的扩展字段，根据TLS协议的握手过程，参与TLS的服务器或者客户端证书应该具有至少两个密钥用法，即数字签名和密钥协商。<br/>
在严格的实现上，缺少任何一个都将导致TLS握手失败。<br/>

##引申阅读
###wiki
[Transport Layer Security](http://en.wikipedia.org/wiki/Transport_Layer_Security)。
###Java
[JSSE关于TLS的概述](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/JSSERefGuide.html\#SSLOverview)。<br/>
[JSSE关于TLS握手日志的一个详尽的例子](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/ReadDebug.html)。<br/>
[关于Java环境TLS问题诊断的一篇blog](https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https)。<br/>
###百度公司
[大型网站的https实践系列](http://op.baidu.com/2015/04/https-index/)。


<br/>
2015-6-8
