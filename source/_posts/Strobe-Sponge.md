---
title: Strobe protocol framework分析 - Sponge Construction
date: 2018-12-28 16:33:28
tags:
- cryptography
- strobe protocol framework
- keccak
mathjax: true
---

[Strobe](https://strobe.sourceforge.io/)是一个面向物联网设计的密码学协议框架，目标是为了使加密协议更容易开发，部署，分析，并适用于微型物联网设备。在Strobe的设计中，仅使用了一个块函数Keccak-f来对消息进行加密和验证。

Strobe利用了Keccak的海绵结构（Sponge Construction）来设计。这使得在Strobe的基础上，可以实现包括密码学哈希函数（Cryptographic hash function），消息认证码（Message authentication code）等机制；可以构建对称加密机制，数字签名机制（通过Schnorr Scheme），密钥交换机制（通过DH算法），以及TLS等类似的密码协议。

在这里将会分析Strobe协议的组成以及其原理，以及如何在Strobe上构建密码协议。

在分析Strobe协议之前，首先需要明确什么是海绵结构。

<!-- more -->

1. [Strobe protocol framework - Sponge Construction](https://tiannian.github.io/2018/12/28/strobe-sponge/)

海绵结构（Sponge Construction）是Keccak提出的一种密码学函数。最开始的海绵结构是为构造密码学哈希函数所设计。后来发现通过海绵结构不仅仅适用于具有固定输出长度的密码学哈希函数，同样也适用于具有固定输入长度的流密码机制。

现有的的哈希函数可以将任意长度的输入映射为固定长度输出（实际上包括SHA2在内的密码学哈希函数也都做不到任意长度的输入，只不过是因为现有的密码学哈希函数可容纳的输入长度上限太大，近似的看作输入长度无限），而海绵结构则是可以将任意长度的输入映射为任意长度的输出。同时它的输出就像是一个随机数预言机（Random Oracle），能够保证输出的随机性。

## Cryptographic Sponge Functions

说了这么多，到底什么是海绵结构呢？

在密码学中的“海绵”，比较正式的说法应该是密码学海绵函数（Cryptographic Sponge Functions）。这个函数就像是一个海绵一样，能够不断，多次的输入来自外部的数据；同时也能够像挤海绵里的水一样，多次向外输出数据。不同于现实中的海绵，这里的海绵是可以无限的输入数据，也可以无限的输出数据。

海绵结构是一个简单的迭代的结构。构造一个海绵结构需要两个元素：一个长度为$b$的状态$S$（state），在状态上展开迭代过程；以及一个用于操作状态，输入和输入长度都为$b$的函数$f$。

![](f.png)

实际上这里的函数$f$是一个纯函数，它接收状态作为传入，返回的是经过它处理后的状态。

而被传入函数$f$的状态$S$被分为两部分，分别是长度为$r$的公共部分$R$（rate），和长度为$c$的私有部分$C$（capacity）。

![](split.png)

