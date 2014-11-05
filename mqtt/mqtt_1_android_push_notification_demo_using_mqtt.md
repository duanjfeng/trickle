mqtt学习 (1)
#基于mqtt的android消息推送demo
`mqtt` `push notification` `消息推送` `java` `android`
##mqtt是什么
mqtt是一个客户端、服务器端之间，基于发布/订阅的消息传输协议。<br>
此协议的设计轻量、开放、简单、易于实现。这些特性使得它适于很多低功耗、带宽昂贵的场景，包括受限环境（例如：机器与机器间通信，M2M）、物联网环境（IoT）。（译自[mqtt官网](http://mqtt.org/)。)
##准备message broker（消息代理）
使用mqtt协议，我们需要准备一个message broker。<br>
mqtt的官网提供了一些开放的[message broker](github.com/mqtt/mqtt.github.io/wiki/public_brokers)。<br>
在这里使用其中的iot.eclipse.org。地址iot.eclipse.org，port为1883，topics有很多，请参看eclipse.mqttbridge.com，这里使用“/test/”作为测试的topic。
##准备android client
github有一个[android client demo](https://github.com/tokudu/AndroidPushNotificationsDemo)，稍微改造一下，以使用上面的message broker。<br>
1.修改PushService的MQTT_HOST属性值为“iot.eclipse.org”。<br>
2.修改MQTTConnection(String brokerHostName, String initTopic)函数中initTopic的取值，由
```java
initTopic = MQTT_CLIENT_ID + "/" + initTopic;
```
调整为
```java
initTopic = "/test/";
```
。
##测试demo
将android client打包部署到android设备上。<br>
运行应用，点击“Start Push service”按钮，几秒后会有消息提示。<br>

2014-11-5
