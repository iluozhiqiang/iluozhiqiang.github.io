---
title: "Go实现websocket协议"
date: 2019-04-18T17:42:34+08:00

lastmod: 2019-04-18T17:42:34+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: []  # 标签
categories: ["index"]              # 分类
author: "大菠萝"                  # 作者

# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: false       # 关闭文章目录
reward: false	 # 关闭打赏
mathjax: false    # 打开 mathjax
---





[WebSocket](http://websocket.org/) 是一种网络通信协议，很多高级功能都需要它。

本文介绍 WebSocket 协议的实现及通信协议。



## 简介

WebSocket 协议在2008年诞生，2011年成为国际标准。所有浏览器都已经支持了。

它的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于[服务器推送技术](https://en.wikipedia.org/wiki/Push_technology)的一种。

![img](https://i.loli.net/2021/05/18/RM2qtDT4PL5yOxE.png)



Websocket既然是建立在TCP/IP协议之上的应用协议，那么就会存在数据传输的协议，而Websocket是借了Http协议的通道升级成一个长链接的Socket链接。



既然是和Socket连接一样，我们就去动手实现一下吧。提高提高我们对通信协议的理解及字节流的操作能力。



## 协议定义

协议地址[rfc6455](https://datatracker.ietf.org/doc/html/rfc6455)，协议直观图。从这张图中可以直观的看到每个字节中每个bit(位)所代表的含义。

![image-20210518190427253](https://i.imgur.com/vcJm6Jb.png)

**FIN**

标识是否为此消息的最后一个数据包，占 1 bit



**RSV**

RSV1, RSV2, RSV3: 用于扩展协议，一般为0，各占 1 bit



**opcode**

> 数据包类型（frame type），占4bits
> 0x0：标识一个中间数据包
>
> 0x1：标识一个text类型数据包
>
> 0x2：标识一个binary类型数据包
>
> 0x3-7：保留
>
> 0x8：标识一个断开连接类型数据
>
> 0x9：标识一个ping类型数据包
>
> 0xA：表示一个pong类型数据包
>
> 0xB-F：保留



**MASK**

用于标识PayloadData是否经过掩码处理。如果是1，Masking-key域的数据即是掩码密钥，用于解码PayloadData。客户端发出的数据帧需要进行掩码处理，所以此位是1。



**Payload length**

Payload data的长度，占7bits，7+16bits，7+64bits：

- 如果其值在0-125，则是payload的真实长度。
- 如果值是126，则后面2个字节形成的16bits无符号整型数的值是payload的真实长度。注意，网络字节序，需要转换。
- 如果值是127，则后面8个字节形成的64bits无符号整型数的值是payload的真实长度。注意，网络字节序，需要转换。

这里的长度表示遵循一个原则，用最少的字节表示长度（尽量减少不必要的传输）。举例说，payload真实长度是124，在0-125之间，必须用前7位表示；不允许长度1是126或127，然后长度2是124，这样违反原则。



**Payload data**
应用层数据



## GO语言实现



**实现步骤**

1. 读取前2个字节
2. 依次解析前2个字节16位的值
3. 通过解析出来的长度读取数据包
4. 如果Masked为true，需要解码数据包



```go
const (
	finalBit = 1 << 7
	rsv1Bit  = 1 << 6
	rsv2Bit  = 1 << 5
	rsv3Bit  = 1 << 4

	maskBit = 1 << 7
)
```



这里先通过左移的方式计算一个值，然后将该值与对应的字节进行&预算，就可以得到该位的值。

这里拿finalBit作为例子讲解下就明白了。

1<<7 该值其实是2^7=128，用bit描述的话就是1000 0000。和读取第一个byte进行与运算，得到的结果如果不为0就表示finnalBit为1，否则的话finalBit就是0。

通过位的位移及与运算，我们可以知道一个字节中任何一位的值是否为1。