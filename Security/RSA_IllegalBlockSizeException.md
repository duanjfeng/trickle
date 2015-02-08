#RSA IllegalBlockSizeException
`java` `RSA` `IllegalBlockSizeException`

##问题现象
java语言执行RSA加解密运算时抛出异常IllegalBlockSizeException。
错误的代码是参考《Java加密与解密的艺术》一书第2版8.3.3节的示例代码写的，网上很多地方能下载，这里仅贴出出错的部分。
```java
cipher.init(Cipher.DECRYPT_MODE, privateKey);
return cipher.doFinal(data);
```
错误堆栈如下：
```java
javax.crypto.IllegalBlockSizeException: Data must not be longer than 53 bytes
	at com.sun.crypto.provider.RSACipher.doFinal(RSACipher.java:337)
	at com.sun.crypto.provider.RSACipher.engineDoFinal(RSACipher.java:382)
	at javax.crypto.Cipher.doFinal(Cipher.java:1970)
```

##原因分析
###IllegalBlockSizeException是何方神圣？
```java
IllegalBlockSizeException - if this cipher is a block cipher, no padding has been requested (only in encryption mode), and the total input length of the data processed by this cipher is not a multiple of block size; or if this encryption algorithm is unable to process the input data provided. 
```
块密码，未使用padding（加密模式下），待处理的数据不是块大小的整数倍；或者算法无法处理输入。
###是不是“待处理的数据不是块大小的整数倍”？
答案为否。拿一个整数部块大小的输入试试，依然报错。原来本质在于“Data must not be longer than 53 bytes”这句话。原来这个异常是属于“算法无法处理输入”那种。

##问题处理
###块大小为多少
块密码的块大小与实现、密钥长度有关。以BouncyCastleProvider实现为例，密钥长度512的话，块大小为63/64；密钥长度1024的话，块大小为117/128。
居然是变化的，这下程序怎么写，固定好实现与密钥长度，然后写两个常量。这是一种办法，但是不太好，如果想换个实现或者密钥长度，还得改常量。
有没有办法动态获取呢？
###动态获取块大小
功夫不负有心人，Cipher类提供了一个getBlockSize方法。可是一试，你就会发现，java返回的都是0，默认的Provider们居然不实现。
###谁实现了getBlockSize
BouncyCastleProvider，BC实现了getBlockSize，所以如果你用BC，那么上述问题迎刃而解，不需要再定义常量来指定块大小了。

##示例代码
问题已经分析清楚了，下面贴出示例代码。
###添加Provider
增加BouncyCastle的jar包后
方法一：修改jre\lib\security\java.security文件添加。<br>
方法二：动态添加，代码如下：
```java
Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
```
###getInstance
所有实例均使用BC提供的实现，即KEY_ALGORITHM为RSA，PROVIDER_NAME为BC。
```java
keyFactory = KeyFactory.getInstance(KEY_ALGORITHM, PROVIDER_NAME);
cipher = Cipher.getInstance(keyFactory.getAlgorithm(), PROVIDER_NAME);
KeyPairGenerator keyPairGen = KeyPairGenerator.getInstance(KEY_ALGORITHM, PROVIDER_NAME);
```
###doCipher
```java
 private static byte[] doCipher(ByteArrayInputStream is, Cipher cipher) throws IOException,
            IllegalBlockSizeException, BadPaddingException {
        byte[] inputBytes = new byte[BUF_SIZE];
        int inputLength = is.read(inputBytes);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int maxBlockSize = cipher.getBlockSize();
        int offset = 0;
        byte[] tempOutput;
        int round = 0;
        while (inputLength > offset) {
            if (inputLength - offset < maxBlockSize) {
                tempOutput = cipher.doFinal(inputBytes, offset, inputLength - offset);
            }
            else {
                tempOutput = cipher.doFinal(inputBytes, offset, maxBlockSize);
            }
            baos.write(tempOutput, 0, tempOutput.length);
            round++;
            offset = round * maxBlockSize;
        }
        byte[] outputBytes = baos.toByteArray();
        baos.close();
        return outputBytes;
    }
```

