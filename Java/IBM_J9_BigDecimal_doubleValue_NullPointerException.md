#IBM J9 BigDecimal doubleValue NullPointerException
`J9 VM` `BigDecimal` `NullPointerException`
#问题简述
一个非空BigDecimal对象在调用setScale方法之后调用doubleValue方法，导致空指针异常。
代码如下（b非空）：
```java
b.setScale(scale, round_mode).doubleValue();
```
堆栈如下：
```java
[12/11/12 18:32:15:515 GMT+08:00] 00002c4c SystemErr     R Caused by: java.lang.NullPointerException
[12/11/12 18:32:15:516 GMT+08:00] 00002c4c SystemErr     R 	at java.math.BigDecimal.toString2(BigDecimal.java:7652)
[12/11/12 18:32:15:516 GMT+08:00] 00002c4c SystemErr     R 	at java.math.BigDecimal.toString(BigDecimal.java:7180)
[12/11/12 18:32:15:516 GMT+08:00] 00002c4c SystemErr     R 	at java.math.BigDecimal.doubleValue(BigDecimal.java:4830)
```

#解决方法
* IBM J9虚拟机中BigDecimal类位于vm.jar包中，源码并没有开放。反编译后未发现空指针发生的原因。
* 使用Oracle虚拟机的源码创建一个BigDecimal包（非java.math包路径），在应用代码中import自定义BigDecimal类。
