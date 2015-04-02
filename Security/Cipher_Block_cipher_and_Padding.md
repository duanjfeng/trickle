#Block cipher and Padding
#分组密码和填充
`Cipher` `Block cipher` `Padding` `密码学` `分组密码` `填充`

##引子
本文是一篇科普文。简要说明什么是分组密码、分组密码对密钥、输入的长度有何要求，分组密码为何需要填充。<br/>
本文仅浅显介绍密码学的应用常识，但密码学的本质是数学，密码设计与分析远比本文的内容高升、复杂，充满魅力。<br/>

##分组密码与序列密码
分组密码，又称块密码，block cipher，特点是将输入分成固定大小的分组，每次处理（加密/解密）一个分组，即若干bit。<br/>
序列密码，又称流密码，stream cipher，特点是无需分组，每次处理（加密/解密）一个输入位，即一个bit。<br/>
常用的分组密码有DES，AES，RSA；常用的序列密码有RC2，RC4。<br/>

##分组密码的密钥长度
分组密码通常有确定的密钥长度。而密钥长度的值与具体实现有关。通常使用同一算法加密，密钥的长度越长，加密强度越高。<br/>
Java7支持的默认密钥长度：DES为56位，DESede为168位；AES为128位；RSA则有1024和2048两种。<br/>

##分组密码的分组长度
分组密码每一个分组的长度通常等于密钥的长度。例如128位的AES算法，会将输入分为128位一组，所以如果不填充的话，你的输入必须刚好为128位的整数倍，否则程序会报异常。<br/>

##分组密码的填充
分组密码的分组长度是确定的，但是程序输入的长度通常是不确定的，很难刚好是分组长度的整数倍。因此大部分情况下，我们需要使用填充机制。即在输入后面补充一定的字节，以确保输入的总长度是分组长度的整数倍。<br/>
常见的填充方式有NoPadding，PKCS5Padding，PKCS7Padding，ISO10126Padding等。<br/>

##编程时的注意事项
简单的说：密钥长度、填充一个都不能错。（其实还有个算法模式，例如CBC/ECB/OFB/CFB等等，这里暂不展开。）<br/>
编程前必须清楚的设计好密钥的长度和填充的机制。如果涉及与第三方交互，更是要约定好这些要素。<br/>
下面是一个示例程序，你可以试着改改KEY\_LEN，例如改成112或者168；改改CIPHER\_ALG，例如改成AES/CBC/NoPadding或者AES/ECB/PKCS5Padding。看看程序运行的结果会有何改变。<br/>
```Java

	package security;
	
	import java.security.InvalidKeyException;
	import java.security.Key;
	import java.security.NoSuchAlgorithmException;
	
	import javax.crypto.BadPaddingException;
	import javax.crypto.Cipher;
	import javax.crypto.IllegalBlockSizeException;
	import javax.crypto.KeyGenerator;
	import javax.crypto.NoSuchPaddingException;
	
	public class BlockCipher {
	
	    private static final int   KEY_LEN    = 128;
	    public static final String UTF_8      = "UTF-8";
	    public static final String CIPHER_ALG = "AES/ECB/NoPadding";
	    public static final String KEY_ALG    = "AES";
	
	    public static void main(String[] args) {
	        try {
	            String inputStr = "input";
	            byte[] inputBytes = inputStr.getBytes(UTF_8);
	            byte[] outputBytes = encrypt(CIPHER_ALG, KEY_ALG, KEY_LEN, inputBytes);
	            System.out.println("inputBytes length: " + inputBytes.length + "; outputBytes length: "
	                    + outputBytes.length);
	        }
	        catch (Exception e) {
	            // TODO logger
	            e.printStackTrace();
	        }
	    }
	
	    private static byte[] encrypt(String cipherAlg, String keyAlg, int keyLen, byte[] inputBytes)
	            throws NoSuchAlgorithmException, NoSuchPaddingException, InvalidKeyException, IllegalBlockSizeException,
	            BadPaddingException {
	        Cipher cipher = Cipher.getInstance(cipherAlg);
	        KeyGenerator generator = KeyGenerator.getInstance(keyAlg);
	        generator.init(keyLen);
	        Key key = generator.generateKey();
	        cipher.init(Cipher.ENCRYPT_MODE, key);
	        byte[] outputBytes = cipher.doFinal(inputBytes);
	        return outputBytes;
	    }
	}

```

#填充算法举例
* NoPadding：不填充数据。
* PKCS5Padding:待处理数据每8个字节一分组，PKCS5Padding为一个字节序列，该序列中每个字节的内容为该字节序列的长度。<br/>例如，待处理数据为FF FF FF FF FF FF，共6个字节，则需要填充2个字节，即02，所以PKCS5Padding的结果为FF FF FF FF FF FF 02 02
* PKCS7Padding:PKCS7Padding为一个字节序列，该序列中每个字节的内容为该字节序列的长度。可以看出，PKCS5Padding是PKCS7Padding的一个子集。PKCS7Padding可以处理的分组长度从1到255字节。<br/>假设分组为16个字节，待处理数据为FF FF FF FF FF FF，共6个字节，则需要填充10个字节，即0A，所以PKCS5Padding的结果为FF FF FF FF FF FF 0A 0A 0A 0A 0A 0A 0A 0A 0A 0A。
* ISO10126Padding：PKCS7Padding为一个字节序列，该序列中最后一个字节的内容为该字节序列的长度，其余为随机数。<br/>假设分组为8个字节，待处理数据为FF FF FF FF FF FF，共6个字节，则需要填充2个字节，即02，所以ISO10126Padding的结果为FF FF FF FF FF FF AB 02。

##相关链接
[RSA IllegalBlockSizeException](https://github.com/duanjfeng/trickle/blob/master/Security/RSA_IllegalBlockSizeException.md)

<br/>
2015-4-2