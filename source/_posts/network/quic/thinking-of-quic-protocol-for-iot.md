---
title: 有关QUIC的随笔 - 如何为IoT设备实施QUIC协议
date: 2019-02-16 21:21:56
tags:
  - QUIC
  - thinking
  - Strobe
  - Noise
  - Disco
---

最近翻译了IETF的QUIC协议提案，在翻译的过程中对QUIC协议有了更深入的理解。同时也思考了一些与QUIC的问题，尤其是针对如何在IoT设备环境中实施QUIC协议进行了一些思考。

QUIC协议最开始是Google设计的用来代替TCP的运输层协议。与TCP不同的是，QUIC工作于应用程序中，而并非工作于操作系统内核中。QUIC更加适配HTTP/2，根据Google给出的数据，基于QUIC的HTTP/2性能好于基于TCP的HTTP/2。IETF甚至将HTTP/2 over QUIC称之为HTTP/3。但IETF给出的协议提案与Google的原始实现并不相同，同时也删除了一些功能。

但在阅读QUIC协议提案时，笔者也发现了一些问题。其中最大的问题时QUIC协议是为HTTP通讯而设计。当笔者考虑将QUIC协议实施于IoT环境下时，发现QUIC的实现过于庞大，同时对设备资源需求较高。QUIC本身确实拥有一些适合于物联网环境的特性，本文就来讨论下如何在IoT设备上实施QUIC协议，避免实施QUIC时会遇到的问题。

<!-- more -->

## QUIC协议与IoT

### 为什么QUIC协议是IoT设备的良伴

### 为什么难以在IoT设备上实施QUIC协议

## 如何修改QUIC协议

### QUIC的逻辑结构

### 寻找TLS的替代品

### 寻找Cubic的替代品

### 阉割QUIC

### 可以选择的应用层协议



