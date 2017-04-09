# TLS握手过程
服务器与客户端商议通信密钥的过程称为TLS握手，在握手阶段，通信内容虽然都是明文，但最终要保证所商量的结果只有客户端和服务端知道，其他中间节点无从得知。  

## RSA握手过程
![](https://blog.cloudflare.com/content/images/2014/Sep/ssl_handshake_rsa.jpg)

RSA握手过程说明：
1. "Client Hello"，客户端向服务器发送一个随机数R1。
2. "Server Hello"，服务器给客户端回复一个随机数R2与包含公钥的证书。
3. "Client Key Exchange"，客户端用公钥加密一个随机数(premaster secret)发送给服务器。
  
```javascript
// 生成master secret、以及从master secret中得到session key
masterSecret = generateMasterSecret(R1, R2, premasterSecret);
sessionKey = sessionKeyFrom(masterSecret);
```  
第3步，premaster secret由客户端使用公钥加密后发送，拥有私钥的服务器才能解密，所以最终只有客户端与服务端能生成一致的master secret。上图中session key是从master serect中派生出来用做对称加密算法的密钥，握手阶段结束后，通信内容开始使用密钥加密并签名，确保有同样知道密钥的人才能解密内容，从而避免被窃听和篡改的风险。  

## 安全问题  
上述握手过程，在私钥没有泄漏且证书是可信的前提下，的确能避免通信内容被窃听或篡改。但是可能存在一种情况：  

> 通信过程中，客户端和服务器之间的某个网络节点，将所有与服务器相关通信内容（TCP报文）在本地保存一份副本。在当时，由于不知道服务器私钥，无法破解被加密的premaster secret，从而不能解密通信内容。很久以后的某天，服务器的私钥泄漏，导致加密的premaster secret被破解，通信内容被解密，从而引发信息泄漏。  

上述RSA握手过程，服务器长期使用的私钥泄漏会导致会话密钥泄漏，从而导致使用会话密钥加密的内容泄漏，即不具有[前向安全性](https://zh.wikipedia.org/wiki/%E5%89%8D%E5%90%91%E5%AE%89%E5%85%A8%E6%80%A7)，下面介绍一种具有前向安全性的握手过程-DHE_RSA。  

## DHE_RSA握手过程
![](https://blog.cloudflare.com/content/images/2014/Sep/ssl_handshake_diffie_hellman.jpg)
  
DHE_RSA握手过程依赖密钥交换算法-[迪菲-赫尔曼密钥交换](https://zh.wikipedia.org/zh-cn/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E5%B0%94%E6%9B%BC%E5%AF%86%E9%92%A5%E4%BA%A4%E6%8D%A2)(Diffie–Hellman key exchange，缩写D-H)，它是一种安全协议，可以让双方在完全没有对方任何预先信息的条件下通过不安全信道创建起一个密钥。  

> DH算法简要原理摘录：
> -   person a has secret a, sends g<sup>a</sup> to person b
> -   person b has secret b, sends g<sup>b</sup> to person a
> -   person a computes (g<sup>b</sup>)<sup>a</sup>
> -   person b computes (g<sup>a</sup>)<sup>b</sup>
> -   Both person a and b end up with g<sup>ab</sup>, which is their shared secret

DHE_RSA握手过程说明：
1. "Client Hello"，客户端向服务器发送一个随机数R1。
2. "Server Hello"，服务器给客户端回复一个随机数R2与包含公钥的证书。
3. "Server Key Exchange"，服务端挑选一个server-DH参数(g<sup>a</sup>)，发送给客户端，并且包含私钥对内容的签名，客户端可用第2步证书中的公钥验证内容未被篡改。
4. "Client Key Exchange"，客户端挑选一个client-DH参数(g<sup>b</sup>)，发送给服务端。

第4步后，双方都能计算出premaster secret(g<sup>ab</sup>)，从而得到master secret以及session key。在这种方式下，服务器私钥只在第3步中签名使用，哪怕泄漏，两个DH参数也不会被破解。  

虽然网上大部分文章讲LTS握手都是第一种RSA方式，但现实中单纯使用RSA握手的TLS几乎没有。使用最多的应该是ECDHE_RSA，原理与DHE_RSA无大区别，借助椭圆曲线(Elliptic Curve)减少计算量，优化了性能。  
  
tips: 通过chrome调试窗口"Security"标签页，可以看到握手过程的具体算法，如："a strong key exchange (ECDHE_RSA with P-256)"。  

待确认：
-   server-DH参数应临时生成，保证不同会话之间安全性不会相互影响，但服务器和客户端可能为追求性能而缓存DH参数，从而导致不同会话的安全性。  
-   除了DHE_RSA，还有一种DH_RSA握手过程，server-DH参数固定在服务器私钥中，一旦私钥泄漏，之前的流量都可解密，不具有前向安全性。  