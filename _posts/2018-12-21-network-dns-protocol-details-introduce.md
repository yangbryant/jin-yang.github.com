---
title: DNS 协议详解
layout: post
comments: true
language: chinese
category: [linux,misc]
keywords: DNS
description: 也就是做 DNS 解析时，客户端和服务端的通讯协议，详见 RFC1035 4.MESSAGES 部分。这里简单介绍其基本概念。
---

也就是做 DNS 解析时，客户端和服务端的通讯协议，详见 [RFC1035 4.MESSAGES](https://www.ietf.org/rfc/rfc1035.txt) 部分。

这里简单介绍其基本概念。

<!-- more -->

## DNS 协议

DNS 协议建立在 UDP 或 TCP 协议之上，默认使用 53 号端口。客户端默认通过 UDP 协议进行通讯，但是由于广域网中不适合传输过大的 UDP 数据包，因此规定当报文长度超过了 512 字节时，应转换为使用 TCP 协议进行数据传输。

可能会出现如下的两种情况：

* 客户端认为 UDP 响应包长度可能超过 512 字节，主动使用 TCP 协议；
* 客户端使用 UDP 协议发送 DNS 请求，服务端发现响应报文超过了 512 字节，在截断的 UDP 响应报文中将 TC 设置为 1 ，以通知客户端该报文已经被截断，客户端收到之后再发起一次 TCP 请求。

### 报文格式

整个报文的格式如下，包括了五部分组成。

{% highlight text %}
    +---------------------+
    |        Header       | 报文头
    +---------------------+
    |       Question      | 查询请求
    +---------------------+
    |        Answer       | 应答
    +---------------------+
    |      Authority      | 授权应答
    +---------------------+
    |      Additional     | 附加信息
    +---------------------+
{% endhighlight %}

详细介绍如下。

* Header 必选，定义了报文是请求还是应答、错误码以及其它的一些标志位；
* Question 描述了查询的请求报文，包括查询类型(QTYPE)、查询类(QCLASS) 以及查询的域名(QNAME)；


剩下的3个段包含相同的格式:一系列可能为空的资源记录(RRs)。
Answer段包含回答问题的RRs；授权段包含授权域名服务器的RRs；
附加段包含和请求相关的，但是不是必须回答的RRs。

#### 报文头

{% highlight text %}
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      ID                       |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |QR|  Opcode  |AA|TC|RD|RA|   Z    |   RCODE    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    QDCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ANCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    NSCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ARCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
{% endhighlight %}

字段介绍如下。

* ID 16Bits 客户端设置，响应报文会原样带回，用于客户端区分不同的请求应答；
* QR 1Bits 区分是请求 (0 Question) 还是应答 (1 Response)；
* Opcode 4Bits 设置查询的种类，响应报文会原样带回，可以为：0 标准查询 QUERY；1 反向查询 IQUERY；2 服务器状态查询 STATUS；3~15 保留；
* AA 1Bits 授权应答 AuthoritativeAnswer，响应报文生效，用于标示服务器响应报文是否为授权服务器返回的结果，可能是在本地 Cache 的缓存；
* TC 1Bits 截断 TrunCation，报文因为超过了允许的长度，导致被截断；
* RD 1Bits 用于请求中，期望使用递归查询；
* RA 1Bits 用于响应中，如果服务器支持递归查询则设置为 1 ，否则设置为 0 ；
* RCODE 4Bits 应答码 ResponseCode，会在响应报文中设置，其代表的含义如下：
	* 0 没有错误；
	* 1 报文格式错误(Format Error)，服务器解析请求报文时报错；
	* 2 服务器失败(Server Failure)，因为服务器的原因导致没办法处理这个请求；
    * 3 名字错误(Name Error)，只对授权域名解析服务器有意义，解析的域名不存在；
	* 4 没有实现(Not Implemented)，域名服务器不支持查询类型；
	* 5 拒绝(Refused)，由于服务器设置的策略拒绝给出应答，通常是安全的配置；
	* 6-15 保留值，暂未使用。
* QDCOUNT 无符号16位整数表示报文请求段中的问题记录数；
* ANCOUNT 无符号16位整数表示报文回答段中的回答记录数；
* NSCOUNT 无符号16位整数表示报文授权段中的授权记录数；
* ARCOUNT 无符号16位整数表示报文附加段中的附加记录数。

#### 查询请求

用来标识，查询的请求参数，同时需要在头中设置 `QDCOUNT` 这个字段。

{% highlight text %}
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                     QNAME                     /
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QTYPE                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QCLASS                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
{% endhighlight %}

字段含义如下:

* QNAME 域名被编码为一些labels序列，每个labels包含一个字节表示后续字符串长度，以及这个字符串，以0长度和空字符串来表示域名结束。注意这个字段可能为奇数字节，不需要进行边界填充对齐。
* QTYPE 2个字节表示查询类型，.取值可以为任何可用的类型值，以及通配码来表示所有的资源记录。
* QCLASS 2个字节表示查询的协议类，比如，IN代表Internet

## 其它

### Non-authoritative answer

为加快 DNS 的查询速度，一般会在服务端缓存一段时间，所以有可能 DNS 会返回缓存在 Cache 中的内容，那么此时就会将 AA 响应设置为 0 ，也就是是这里显示的 Non-authoritative answer 。

### 递归查询 VS. 迭代查询

在递归查询模式下，DNS 服务器在接收到客户机请求时，必须使用一个准确的查询结果回复客户机。也就意味着，如果 DNS 服务器本地没有缓存所查询的 DNS 信息，那么该服务器会询问其它服务器，并将返回的查询结果提交给客户机。

而在使用迭代查询时，DNS 服务器会向客户机提供其它能够解析查询请求的 DNS 服务器地址。也就是说，当客户机发送查询请求时，DNS 服务器并不直接回复查询结果，而是告诉客户机另一台 DNS 服务器地址，客户机需要再向这台 DNS 服务器提交请求，依次循环直到返回查询的结果为止。

也就是说，关键的区别是由谁去查询最终的结果。

### DDoS

发送大量的 DNS 递归查询会消耗服务端的一定资源，所以，只需要将发送的报文设置一个 RD 标志位即可。

当发送垃圾查询时，例如 `www.baidu.com\n` 这类必然不存在的域名，必然会导致查询很慢。

## 参考

DNS 协议 [RFC1035](https://www.ietf.org/rfc/rfc1035.txt) 详细规定了 DNS 报文的格式，详见 `4. MESSAGES` 中的部分。

{% highlight text %}
{% endhighlight %}
