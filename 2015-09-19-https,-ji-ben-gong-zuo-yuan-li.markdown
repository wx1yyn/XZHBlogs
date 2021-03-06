---
layout: post
title: "HTTPS、基本工作原理"
date: 2015-09-19 11:12:32 +0800
comments: true
categories: 
---

###HTTPS 相当于是 `HTTP over SSL`

关于这句话解释可以在 SSL与TLS 章节找到.


***

###HTTPS和HTTP的区别

- https协议 需要一个`SSL证书`，一般免费证书很少，需要交费

- http是超文本传输协议，信息是`明文传输`

- https 则是具有安全性的`SSL加密`传输协议

- http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443

- http的连接很简单，是无状态的

- HTTPS协议是由`SSL+HTTP协议`构建的可进行加密传输、身份认证的网络协议，比http协议安全

####最大的区别

- 使用 `https` 进行传输数据时
- 最终 `应用层` 仍然是使用 `http协议`
- 只是不会像以前直接 `应用层 ----> 传输层`
- 而是经过SSL层处理，`应用层 -----> SSL层 -----> 传输层`
- SSL层负责对传输数据的`加密、完整性验证、双方身份认证`

****

###从网络截了一个客户端与服务端使用`https协议`交互过程图示

![](http://i2.piimg.com/7dbf2dc093a50642.jpg)

***

###浏览器（客户端）打开一个https协议服务器域名时 大概做的事情:

- `浏览器` 将自己支持的 所有加密规则与HASH算法 发送给 服务器网站

- 服务器网站接收到 所有的加密规则之后 做的事情:
	
	- 从中选出 一组加密规则与HASH算法
	- 将自己的 身份信息 以 `证书的形式` 发回给浏览器。证书包含:
		- 网站地址
		- 加密公钥（申请的证书 公钥）
		- 证书的颁发机构等

- 浏览器获得 服务器发送的证书之后，做的事情:

	- 验证接收的证书是否合法

		- 证书颁发机构是否被信任的
		- 证书中包含的网站地址 是否与 正在访问的地址 一致等
		- 如果证书受信任，则浏览器栏里面会显示一个 小锁头
		- 否则会弹出证书不受信的提示

	- 如果证书合法 或 用户同意使用不被信任的证书
		- 浏览器会生成一串 随机数 的密码
		- 并用 证书中提供的 公钥 对 随机数 加密

	- 使用约定好的HASH算法 计算 握手消息，并使用生成的 `随机数` 对消息进行加密，然后发送给 服务器网站

- 服务器网站 接收浏览器发来的数据之后要做的事情:

	- 使用自己的 `私钥` 将信息解密取出 加密的随机数
	- 使用 解密出来的随机数 解密浏览器发来的 握手消息
	- 然后使用之前从浏览器选择的 `HASH算法` 计算 握手信息 
	- 并验证 HASH计算出来的握手信息 是否与 浏览器发来的握手信息 一致
	- 其实就是验证 浏览器的HASH值 与 服务器的HASH值 是否一致
		- 都是对握手信息进行HASH
	- 最后又使用 随机数加密 握手信息，发回给浏览器

- 浏览器解密并计算握手消息的HASH，如果与服务端发来的HASH一致，此时握手过程结束，之后所有的通信数据将由之前浏览器生成的随机密码并利用对称加密算法进行加密.

***

###小结上述过程:

- 浏览器（客户端）与 服务器 之间，是通过 `一个证书` 进行连接以及数据传输.

- 浏览器与服务器网站，互相发送 加密的握手消息 ，然后通过 `随机数密码` 解密并验证其HASH值是否一致
	- HASH值，通常检验一个东西是否发生改变

- 目的是为了保证双方都获得了 `一致的随机数密码`，并且可以正常的加密解密数据，为后续真正数据的传输做一次测试

- `加密`算法，保证传输数据的 `安全`，使用密文传输.

- `HASH`算法，检验一个东西 是否被 `修改 或 非法篡改`.

****

###HTTPS一般使用的 `加密` 与 `HASH算法` 如下:

- 非对称加密算法
	- RSA
	- DSA/DSS

- 对称加密算法
	- AES
	- RC4
	- 3DES

- HASH算法
	- MD5
	- SHA1
	- SHA256

***

###SSL(TLS)握手过程中如果有任何错误，都会使加密连接断开，从而阻止了隐私信息的传输.

所以说，从`已有证书的建立连接`情况下，进行篡改数据，这种办法已经无法可行.

***

###浏览器（客户端）与 服务器 之间，是通过 `SSL证书` 进行连接以及数据传输

- 所以就可能出现 拦截请求，然后伪造 一个假的证书，发送给 服务端或客户端.

- 但是即便是伪造证书，也只是伪造证书中的 `公钥`

- 证书的 `私钥` 是单独 存放在服务器上的，不会对外公布，更不会在网络上传输

- 而 `公钥` 只能对数据 `加密`，`无法对数据 解密`

- 就算被解密了，那么数据的一些特征码就会编码（HASH值变化），SSL识别到数据被窃取或篡改，就会断开连接

所以只要去受信任的CA证书颁发机构，申请一个SSL证书，然后使用https://进行通信，基本上可以保证数据的安全性.

