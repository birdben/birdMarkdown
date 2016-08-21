---
title: "移动端请求URL的加密/解密"
date: 2015-11-13 00:35:34
tags: [其他]
categories: [其他]
---

### 移动端请求URL的加密方式

最近一直在研究如何爬取手机客户端请求的数据信息，发现很多手机客户端在请求服务端数据的时候，对请求的URL都做了一些加密处理，所以自己就私下里研究了一下URL的一些加密算法

下面是一个比较基本的URL加密处理方式：

- 1.服务端和客户端约定好一个公约KEY
- 2.选择一种加密方式MD5，SHA等等（这里重点介绍Mac算法）
- 3.选择要做加密处理的参数（一般是静态参数 + 动态参数的方式，比如：loginToken登录秘钥 + timestamp时间戳）
- 4.客户端根据公约KEY使用Mac算法，对指定的参数（loginToken + timestamp）进行加密处理生成秘钥secret，把secret当做参数传递给服务端
- 5.服务端会也会根据公约KEY使用Mac算法，对指定的参数做同样的加密处理，并且把结果和secret参数进行比较，来判断这个请求是否是正确的

参数：
> loginToken : 8d940b09872346a****0f9844c70d51

> timestamp : 1447327804725

客户端加密处理的秘钥：
> secret : 8d940b098773464796b****844c70d516447327804725


### MAC算法（这里的MAC可不是苹果笔记本。。）

MAC算法结合了MD5和SHA算法的优势，并加入密钥的支持，是一种更为安全的消息摘要算法。

MAC（Message Authentication Code，消息认证码算法）是含有密钥的散列函数算法，兼容了MD和SHA算法的特性，并在此基础上加入了密钥。

MAC算法主要集合了MD和SHA两大系列消息摘要算法。MD系列的算法有HmacMD2、HmacMD4、HmacMD5三种算法；SHA系列的算法有HmacSHA1、HmacSHA224、HmacSHA256、HmacSHA384.HmacSHA512五种算法。

经过MAC算法得到的摘要值也可以使用十六进制编码表示，其摘要值长度与参与实现的摘要值长度相同。例如，HmacSHA1算法得到的摘要长度就是SHA1算法得到的摘要长度，都是160位二进制，换算成十六进制编码为40位。


#### 模型分析

甲乙双方进行数据交换可以采取如下流程完成

- 1. 甲方向乙方公布摘要算法（就是指定要使用的摘要算法名）
- 2. 甲乙双方按照约定构造密钥，双方拥有相同的密钥（一般是一方构造密钥后通知另外一方，此过程不需要通过程序实现，就是双方约定个字符串，但是这个字符串可不是随便设定的，也是通过相关算法获取的）
- 3. 甲方使用密钥对消息做摘要处理，然后将消息和生成的摘要消息一同发送给乙方
- 4. 乙方收到消息后，使用甲方已经公布的摘要算法+约定好的密钥 对收到的消息进行摘要处理。然后比对自己的摘要消息和甲方发过来的摘要消息。甄别消息是否是甲方发送过来的

#### MAC用于消息认证

![](http://static.oschina.net/uploads/space/2014/0714/165313_7RfC_1469576.png)

#### 消息认证码

密码学中，通信实体双方使用的一种验证机制，保证消息数据完整性的一种工具。

安全性依赖于Hash函数，故也称带密钥的Hash函数。

消息认证码是基于密钥和消息摘要【hash】所获得的一个值，目的是用于验证消息的完整性，确认数据在传送和存储过程中未受到主动攻击

![](http://static.oschina.net/uploads/img/201407/14165727_ohv6.png)

下面是使用HmacSHA1进行加密的具体代码

```
/**
 * 初始化HmacSHA1密钥
 *
 * @return byte[] 密钥
 * @throws Exception
 */
public static byte[] initHmacSHAKey() throws Exception {
    // 初始化KeyGenerator
    KeyGenerator keyGenerator = KeyGenerator.getInstance("HmacSHA1");
    // 产生密钥
    SecretKey secretKey = keyGenerator.generateKey();
    // 获得密钥
    return secretKey.getEncoded();
}
 
/**
 * HmacSHA1消息摘要
 *
 * @param data 待做摘要处理的数据
 * @param key  密钥
 * @return byte[] 消息摘要
 * @throws Exception
 */
public static byte[] encodeHmacSHA(byte[] data, byte[] key)
        throws Exception {
    // 还原密钥
    SecretKey secretKey = new SecretKeySpec(key, "HmacSHA1");
    // 实例化Mac
    Mac mac = Mac.getInstance(secretKey.getAlgorithm());
    // 初始化Mac
    mac.init(secretKey);
    // 执行消息摘要
    return mac.doFinal(data);
}
```


参考文章：

- http://my.oschina.net/xinxingegeya/blog/290594
- http://blog.csdn.net/kongqz/article/details/6281710