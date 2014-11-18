mqtt学习 (2)
#mqtt控制报文格式-固定头-第1个字节
`mqtt` `packet format` `报文格式`
##小广告
mqtt v3.1.1已经成为oasis的正式标准啦：[oasis公告](https://www.oasis-open.org/news/announcements/mqtt-version-3-1-1-becomes-an-oasis-standard)，[mqtt v3.1.1标准html版](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html)，[mqtt v3.1.1标准pdf版](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.pdf)。
##mqtt控制报文格式-固定头-类型
第1个字节的第7~4位，4位无符号值。
<table>
<tbody>
<tr><td><em>名称</em></td><td><em>取值</em></td><td><em>方向</em></td><td><em>描述</em></td></tr>
<tr><td>保留</td><td>0</td><td>禁用</td><td>保留</td></tr>
<tr><td>CONNECT</td><td>1</td><td>C->S</td><td>客户端连接服务器请求</td></tr>
<tr><td>CONNACK</td><td>2</td><td>S->C</td><td>请求连接确认</td></tr>
<tr><td>PUBLISH</td><td>3</td><td>C->S或者S->C</td><td>发布消息</td></tr>
<tr><td>PUBACK</td><td>4</td><td>S->C</td><td>发布消息确认</td></tr>
<tr><td>PUBREC</td><td>5</td><td>S->C</td><td>发布消息收到（确认送达第1部分）</td></tr>
<tr><td>PUBREL</td><td>6</td><td>S->C</td><td>发布消息告知（确认送达第2部分）</td></tr>
<tr><td>PUBCOMP</td><td>7</td><td>S->C</td><td>发布消息完成（确认送达第3部分）</td></tr>
<tr><td>SUBSCRIBE</td><td>8</td><td>C->S</td><td>客户端订阅请求</td></tr>
<tr><td>SUBACK</td><td>9</td><td>S->C</td><td>订阅请求确认</td></tr>
<tr><td>UNSUBSCRIBE</td><td>10</td><td>C->S</td><td>取消订阅请求</td></tr>
<tr><td>UNSUBACK</td><td>11</td><td>S->C</td><td>取消订阅确认</td></tr>
<tr><td>PINGREQ</td><td>12</td><td>C->S</td><td>PING请求</td></tr>
<tr><td>PINGRESP</td><td>13</td><td>S->C</td><td>PING响应</td></tr>
<tr><td>DISCONNECT</td><td>14</td><td>C->S</td><td>客户端断开连接</td></tr>
<tr><td>Reserved</td><td>15</td><td>禁用</td><td>保留</td></tr>
</tbody>
</table>
##mqtt控制报文格式-固定头-标志位
第1个字节的第3~0位。对于标志位“保留”的报文，第3~0位取值必须严格按照下表填写，否则接受者会断开网络连接。
<table>
<tbody>
<tr><td><em>控制报文类型</em></td><td><em>标志位</em></td><td><em>Bit 3</em></td><td><em>Bit 2</em></td><td><em>Bit 1</em></td><td><em>Bit 0</em></td></tr>
<tr><td><em>CONNECT</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>CONNACK</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PUBLISH</em></td><td><em>MQTT 3.1.1使用</em></td><td><em>DUP</em></td><td><em>QoS</em></td><td><em>QoS</em></td><td><em>RETAIN</em></td></tr>
<tr><td><em>PUBACK</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PUBREC</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PUBREL</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PUBCOMP</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>SUBSCRIBE
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>SUBACK
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>UNSUBSCRIBE
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>UNSUBACK
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PINGREQ
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>PINGRESP
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
<tr><td><em>DISCONNECT
</em></td><td><em>Reserved</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td><td><em>0</em></td></tr>
</tbody>
</table>
* DUP：Duplicate delivery of a PUBLISH Control Packet，重复投递发布报文
* QoS：PUBLISH Quality of Service，消息发布的服务质量。占用2位，0为至多一次，1为至少一次，2为仅一次，3保留不用。
* RETAIN：PUBLISH Retain flag，发布消息保留。

2014-11-19
