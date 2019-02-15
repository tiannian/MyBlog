---
title: QUIC协议翻译 - draft-ietf-quic-transport-latest Part 2.
date: 2019-02-14 13:34:06
tags:
  - QUIC
  - translate
  - RFC
  - Protocol
---

本文是对QUIC协议IETF提案：[draft-ietf-quic-transport-latest](https://quicwg.org/base-drafts/draft-ietf-quic-transport.html)翻译的第一部分。

[draft-ietf-quic-transport-latest](https://quicwg.org/base-drafts/draft-ietf-quic-transport.html)是在[draft-ietf-quic-invariants-latest](https://quicwg.org/base-drafts/draft-ietf-quic-invariants.html)基础上定义的第一个版本的QUIC协议，这里描述了QUIC协议的核心，包括一些概念定义与具体的协议行为。

第一部分翻译内容包括第4，5节。

<!-- more -->

### 流量控制

为了防止发送方的速度过快从而导致接收者难以接受数据，或防止恶意发送方在接收方处消耗大量内存，有必要限制接收方可以缓冲的数据量，为了使接收方能够限制对连接的内存，并对发送方实施调控，流被单独地和作为聚合流控制。QUIC接收方控制发送方可以在流上发送的最大数据量，如第4.1节和第4.2节所述。

同样，为了限制连接中的并发性，QUIC端点控制其对等方可以启动的最大累积流数，如第4.5节所述。

在CRYPTO帧中发送的数据的流控制方式与流数据的流控制方式不同。 QUIC依赖于加密协议实现来避免过度缓冲数据，请参阅[QUIC-TLS]。具体的实现中应该为QUIC提供一个接口来告诉QUIC它的缓冲限制，以便在限制不同层次上过多的缓冲。